---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-14
source_count: 1
---

# DeepFool Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L2  
**Objective:** Craft a *targeted* adversarial MNIST image — the model must predict a specific target class and the L2 distance from the baseline must stay within `l2_threshold`.

## Scenario

Unlike the [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]] (L∞, any misclassification), this challenge requires:

1. The adversarial image is classified as a **specific target class** (not just any wrong class).
2. The pixel-space L2 distance from the baseline is ≤ `l2_threshold`.

The approach is a targeted DeepFool iteration: at each step, compute the minimal perturbation to move across the linearized decision boundary toward the target class, accumulate it, and repeat until the model predicts the target.

---

## Phase 0 — Environment Setup

```bash
mkdir deep_fool; cd deep_fool
python3 -m venv venv && source venv/bin/activate
pip3 install numpy requests torch torchvision pillow
```

---

## Phase 1 — API and Challenge Fetch

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
# {"status":"ok","l2_threshold":0.75,"index":95,"target":6}
```

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Returns `l2_threshold`, sample `index`, `target` |
| `/challenge` | GET | Returns `{sample_index, label, target, l2_threshold, image_b64}` |
| `/predict` | POST | Test prediction; returns `{pred, confidence}` |
| `/weights` | GET | Download model `state_dict` |
| `/submit` | POST | Validate L2 + target class; returns flag |

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
from PIL import Image

BASE_URL  = os.getenv("BASE_URL", "http://127.0.0.1:8000")
MNIST_MEAN, MNIST_STD = 0.1307, 0.3081

def x01_from_b64_png(b64: str) -> np.ndarray:
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0.0, 1.0)  # (28,28)

def b64_png_from_x01(x2d: np.ndarray) -> str:
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img = Image.fromarray(x255, mode="L")
    buf = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

ch        = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x         = x01_from_b64_png(ch["image_b64"])   # (28,28) in [0,1]
label     = int(ch["label"])
target    = int(ch["target"])
threshold = float(ch["l2_threshold"])
print(f"label={label}  target={target}  l2_threshold={threshold}")
```

---

## Phase 2 — Model Architecture and Weight Loading

Same `SimpleClassifier` as the FGSM challenge. Normalization is **external** — the model takes already-normalized tensors; callers apply `mnist_normalize()`.

```python
class SimpleClassifier(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1    = nn.Conv2d(1, 32, 3, 1)
        self.conv2    = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1      = nn.Linear(9216, 128)
        self.fc2      = nn.Linear(128, 10)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = torch.relu(self.conv1(x))
        x = torch.relu(self.conv2(x))
        x = torch.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = torch.relu(self.fc1(x))
        x = self.dropout2(x)
        return torch.log_softmax(self.fc2(x), dim=1)

def mnist_normalize(x01: torch.Tensor) -> torch.Tensor:
    return (x01 - MNIST_MEAN) / MNIST_STD

wt = requests.get(f"{BASE_URL}/weights", timeout=15).content
with open("deepfool_weights.pth", "wb") as f:
    f.write(wt)
model = SimpleClassifier()
model.load_state_dict(torch.load("deepfool_weights.pth", map_location="cpu"))
model.eval()

x_t = torch.from_numpy(x[None, None, ...]).float()
clean_pred = int(torch.argmax(model(mnist_normalize(x_t)), dim=1).item())
print(f"Clean pred: {clean_pred}  (true label: {label})")
```

---

## Phase 3 — Targeted DeepFool Algorithm

DeepFool finds the **minimum L2 perturbation** to cross the decision boundary. The targeted variant steers toward a specific class by computing the linearized boundary between the current predicted class and the fixed target class at each step.

### Algorithm intuition

At each iteration, given current point `x`:
1. Compute the logit for the current predicted class (`logit[pred]`) and for the target class (`logit[target]`).
2. Compute their respective gradients w.r.t. `x`.
3. The decision boundary between `pred` and `target` is a hyperplane with normal `w = ∇logit[target] − ∇logit[pred]` and offset `g = logit[target] − logit[pred]`.
4. The minimum L2 step to cross that boundary is `|g| / ‖w‖`, applied in the direction of `w / ‖w‖`.
5. Accumulate these steps in `r_tot` and add a small overshoot to ensure the quantized PNG remains on the other side.

```python
def deepfool_targeted(
    model: nn.Module,
    x01: np.ndarray,
    target: int,
    overshoot: float = 0.08,
    max_iter: int = 100,
) -> np.ndarray:
    """
    Targeted DeepFool in [0,1] pixel space.

    Parameters
    ----------
    x01 : np.ndarray  shape (1,1,28,28) in [0,1]
    target : int      desired output class
    overshoot : float multiplicative margin; accounts for PNG quantization rounding

    Returns
    -------
    np.ndarray  adversarial image, shape (1,1,28,28) in [0,1]
    """
    x01_t = torch.from_numpy(x01).float()
    r_tot = torch.zeros_like(x01_t)          # accumulated perturbation

    with torch.enable_grad():
        for _ in range(max_iter):
            # Apply current perturbation and re-linearize
            x = (torch.clamp(x01_t + (1 + overshoot) * r_tot, 0.0, 1.0)
                 .detach().requires_grad_(True))

            logits = model(mnist_normalize(x))
            pred   = int(torch.argmax(logits, dim=1).item())
            if pred == target:
                break

            # Gradient of current predicted class score w.r.t. pixel input
            model.zero_grad(set_to_none=True)
            logits[0, pred].backward(retain_graph=True)
            grad_pred = x.grad.detach().clone()
            x.grad.zero_()

            # Gradient of target class score w.r.t. pixel input
            model.zero_grad(set_to_none=True)
            logits[0, target].backward(retain_graph=True)
            grad_t = x.grad.detach().clone()
            x.grad.zero_()

            # Boundary direction and distance
            w     = grad_t - grad_pred                        # boundary normal
            g     = (logits[0, target] - logits[0, pred]).detach()
            denom = torch.norm(w.flatten()) + 1e-12
            p     = torch.abs(g) / denom                     # distance to boundary

            # Minimal step to cross boundary (+ small epsilon for numerical stability)
            r_i   = (p + 1e-4) * w / (torch.norm(w.flatten()) + 1e-12)
            r_tot = r_tot + r_i

    x_adv = torch.clamp(x01_t + (1 + overshoot) * r_tot, 0.0, 1.0)
    return x_adv.detach().cpu().numpy()
```

### Overshoot schedule

A single overshoot value may not survive PNG quantization (8-bit rounding can push the image back across the boundary). Try increasing overshoots until the constraint is satisfied:

```python
overshoots = [0.08, 0.10, 0.12, 0.15, 0.25]
x_adv = None
for ov in overshoots:
    cand = deepfool_targeted(
        model, x[None, None, ...].astype(np.float32), target,
        overshoot=ov, max_iter=100
    )
    cand_pred = int(torch.argmax(
        model(mnist_normalize(torch.from_numpy(cand))), dim=1).item())
    cand_l2   = float(np.linalg.norm(cand - x[None, None, ...]))
    if cand_pred == target and cand_l2 <= threshold:
        x_adv = cand
        print(f"Success at overshoot={ov}: pred={cand_pred}, L2={cand_l2:.4f}")
        break

if x_adv is None:                        # submit best candidate; server will validate
    x_adv = deepfool_targeted(
        model, x[None, None, ...].astype(np.float32), target, overshoot=0.08)
```

---

## Phase 4 — Validate and Submit

```python
l2 = float(np.linalg.norm(x_adv - x[None, None, ...]))
print(f"L2 = {l2:.4f}  (threshold: {threshold})")

adv_pred = int(torch.argmax(
    model(mnist_normalize(torch.from_numpy(x_adv))), dim=1).item())
print(f"Adversarial pred: {adv_pred}  (target: {target})")

b64  = b64_png_from_x01(x_adv[0, 0])
resp = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64}, timeout=15)
resp.raise_for_status()
print(resp.json())
# {"ok": true, "pred": 6, "target": 6, "l2": 0.72, "flag": "HTB{...}"}
```

Validation errors:

| HTTP status | Message | Fix |
|-------------|---------|-----|
| 400 | `"L2 too large: 0.82 > 0.75"` | Increase `overshoot`; decreases step size relative to boundary distance |
| 400 | `"Wrong target: predicted 4, need 6"` | Attack didn't reach target; increase `overshoot` or `max_iter` |
| 400 | `"Image must be 28x28"` | Wrong image dimensions |

---

## Lessons Learned

- **DeepFool vs. gradient descent**: Simple gradient descent + L2 projection works but takes many iterations and doesn't guarantee minimum perturbation. DeepFool's boundary-crossing step directly computes how far you need to move at each linearization, converging faster with a smaller perturbation.
- **Two gradients per iteration**: You need both `∂logit[target]/∂x` and `∂logit[pred]/∂x`. PyTorch requires `retain_graph=True` for the first backward call and clearing gradients between the two calls.
- **Overshoot compensates for PNG quantization**: The server decodes your submission from 8-bit PNG, which rounds pixel values. An overshoot of 0.08–0.15 (8–15% extra perturbation) ensures the quantized image lands on the target side of the boundary.
- **r_tot is accumulated in pixel space**: Each `r_i` step is in pixel space (because `x` has `requires_grad=True` and normalization is external). L2 is measured directly on `x_adv − x_orig` in pixel space.

---

## Related Pages

- [[attack/ai/deepfool]] — full DeepFool theory: linearized boundary, multi-class formulation, ρ_adv robustness metric
- [[attack/ai/fgsm]] — single-step L∞ attack (simpler but less efficient)
- [[attack/ai/i_fgsm]] — iterative L∞ attack; same idea as DeepFool but projected onto L∞ ball
- [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]] — L∞ untargeted variant using same model
- [[labs/htb/ai_evasion_first_order_attacks/skills_assessment_2]] — DeepFool on CIFAR-10, untargeted, L2 in normalized space

## Sources

- raw/lab/ai_evasion_first_order_attacks/deepfool_challenge.md
