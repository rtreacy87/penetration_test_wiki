---
tags: [lab, attack/ai]
module: ai_evasion_sparsity
last_updated: 2026-05-12
---

# AI Evasion Skills Assessment — JSMA Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion & Sparsity  
**Difficulty:** Skills Assessment  
**Goal:** Craft an adversarial MNIST image under an L0 pixel budget using JSMA; submit via API to receive the flag

## Scenario

An API exposes a fixed MNIST digit classifier (LeNet-5 / MNISTClassifier). You receive a baseline image, a target misclassification class, and a maximum number of pixels you may modify (L0 budget). Use JSMA to iteratively select and modify the most salient pixels until the model predicts the target class. Submit the adversarial image to receive the flag.

**Constraints:**
- `‖x_adv − x‖₀ ≤ budget` — at most `budget` pixels modified
- `‖x_adv − x‖₂ ≤ max_l2` — L2 safeguard against replacing the image wholesale
- All pixel values must stay in `[0, 1]`
- The submission must be derived from the provided baseline image (server checks)

---

## Phase 1 — API Setup and Challenge Fetch

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
import torch.nn.functional as F
from PIL import Image

BASE_URL = os.getenv("BASE_URL", "http://INSTANCE_IP:PORT")
MNIST_MEAN, MNIST_STD = 0.1307, 0.3081

# --- Image encoding helpers ---
def x01_from_b64(b64: str) -> np.ndarray:
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.asarray(img, dtype=np.float32) / 255.0  # (28, 28) in [0,1]

def b64_from_x01(x2d: np.ndarray) -> str:
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img = Image.fromarray(x255, mode="L")
    buf = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

def count_modified(a, b, thr=1e-6):
    return int(np.sum(np.abs(a - b) > thr))

# Fetch challenge
ch = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x = x01_from_b64(ch["image_b64"])   # baseline image (28, 28)
orig_label = int(ch["original_label"])
target     = int(ch["target_class"])
budget     = int(ch["l0_budget"])
max_l2     = float(ch["max_l2"])

print(f"original={orig_label}  target={target}  budget={budget}  max_l2={max_l2:.1f}")

# Verify server prediction matches original label
r = requests.post(f"{BASE_URL}/predict",
                  json={"image_b64": b64_from_x01(x)}, timeout=10).json()
assert r["predicted_class"] == orig_label, f"Mismatch: server={r['predicted_class']}"
```

---

## Phase 2 — Load Model Locally

The server exposes the exact model weights so you can compute Jacobians locally rather than via expensive API queries.

```python
class MNISTClassifier(nn.Module):
    """LeNet-5 style: Conv→Pool→Conv→Pool→FC→FC→FC"""
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 6,  kernel_size=5)
        self.conv2 = nn.Conv2d(6, 16, kernel_size=5)
        self.pool  = nn.AvgPool2d(kernel_size=2, stride=2)
        self.fc1   = nn.Linear(16*4*4, 120)
        self.fc2   = nn.Linear(120, 84)
        self.fc3   = nn.Linear(84, 10)
        self.act   = nn.Tanh()

    def forward(self, x):
        x = self.act(self.conv1(x));  x = self.pool(x)   # →(B,6,12,12)
        x = self.act(self.conv2(x));  x = self.pool(x)   # →(B,16,4,4)
        x = torch.flatten(x, 1)
        x = self.act(self.fc1(x));   x = self.act(self.fc2(x))
        return F.log_softmax(self.fc3(x), dim=1)

def mnist_normalize(x01: torch.Tensor) -> torch.Tensor:
    return (x01 - MNIST_MEAN) / MNIST_STD

# Download weights and load
wt = requests.get(f"{BASE_URL}/weights", timeout=30).content
open("jsma_weights.pth", "wb").write(wt)
model = MNISTClassifier().eval()
model.load_state_dict(torch.load("jsma_weights.pth", map_location="cpu"))

# Confirm local prediction matches server
x_t = torch.from_numpy(x[None, None, ...]).float()  # (1,1,28,28)
with torch.no_grad():
    logits = model(mnist_normalize(x_t))
local_pred = int(torch.argmax(logits, dim=1))
print(f"Local pred={local_pred}, server pred={orig_label}")
assert local_pred == orig_label
```

---

## Phase 3 — JSMA Implementation

JSMA selects the pixel with the highest **saliency score**: the score maximises the gradient toward the target class while minimising gradients toward all other classes.

**Saliency score for pixel i:**
```
S(i) = (∂F_target/∂x_i) × |Σ_{j≠target} ∂F_j/∂x_i|
     (only pixels where ∂F_target/∂x_i > 0 and Σ∂F_j/∂x_i < 0)
```

```python
def jsma_attack(model, x_np, target, budget, theta=1.0):
    """
    x_np: (28,28) numpy array in [0,1]
    target: integer class to force
    budget: max pixels to modify (L0)
    theta: perturbation per selected pixel (1.0 = max brightness)
    """
    x_adv = x_np.copy()
    modified = set()

    for step in range(budget):
        # Build tensor with gradient tracking
        x_t = torch.from_numpy(x_adv[None, None, ...]).float().requires_grad_(True)
        logits = model(mnist_normalize(x_t))   # (1, 10) log-softmax
        probs  = torch.exp(logits)             # (1, 10) probabilities

        # Compute Jacobian: ∂prob_c/∂x_pixel for all classes c
        jacobian = torch.zeros(10, 28*28)
        for c in range(10):
            if x_t.grad is not None:
                x_t.grad.zero_()
            probs[0, c].backward(retain_graph=True)
            jacobian[c] = x_t.grad.view(-1)

        # Saliency: target gradient vs. sum of other gradients
        target_grad = jacobian[target]                        # (784,)
        other_grad  = jacobian.sum(0) - target_grad          # (784,)

        # Mask: valid pixels must be modifiable and not yet saturated
        already = torch.zeros(784, dtype=torch.bool)
        for idx in modified:
            already[idx] = True
        at_max = torch.from_numpy(x_adv.flatten()) >= 1.0

        valid = (~already) & (~at_max) & (target_grad > 0) & (other_grad < 0)
        if not valid.any():
            print(f"No valid pixels at step {step}; stopping")
            break

        # Select pixel with highest saliency score
        saliency = torch.zeros(784)
        saliency[valid] = target_grad[valid] * (-other_grad[valid])
        best_pixel = int(torch.argmax(saliency))

        # Perturb
        row, col = divmod(best_pixel, 28)
        x_adv[row, col] = min(1.0, x_adv[row, col] + theta)
        modified.add(best_pixel)

        # Check if attack succeeded
        with torch.no_grad():
            x_check = torch.from_numpy(x_adv[None, None, ...]).float()
            pred = int(torch.argmax(model(mnist_normalize(x_check)), dim=1))
        if pred == target:
            print(f"Success at step {step+1}: pred={pred}, pixels modified={len(modified)}")
            break

    return x_adv, len(modified)

x_adv, pixels_used = jsma_attack(model, x, target, budget)
print(f"Pixels modified: {pixels_used}/{budget}")
```

**Theta guidance:**
- `theta=1.0` — set pixel to maximum brightness immediately; fewest steps needed
- `theta=0.5` — more gradual, may allow tighter L0 usage
- Use `theta=1.0` first; only reduce if you're exceeding the L0 budget

---

## Phase 4 — Validate and Submit

```python
# Validate locally before submitting
with torch.no_grad():
    x_final = torch.from_numpy(x_adv[None, None, ...]).float()
    final_pred = int(torch.argmax(model(mnist_normalize(x_final)), dim=1))

pixels_changed = count_modified(x, x_adv)
l2_dist = float(np.linalg.norm(x_adv - x))
print(f"Final pred={final_pred} (target={target}), L0={pixels_changed}/{budget}, L2={l2_dist:.3f}/{max_l2}")

assert final_pred == target,          "Target not achieved locally"
assert pixels_changed <= budget,      f"L0 violated: {pixels_changed} > {budget}"
assert l2_dist <= max_l2,             f"L2 violated: {l2_dist:.3f} > {max_l2}"

# Verify server prediction
server_r = requests.post(f"{BASE_URL}/predict",
                         json={"image_b64": b64_from_x01(x_adv)}, timeout=10).json()
print(f"Server pred={server_r['predicted_class']}, confidence={server_r['confidence']:.3f}")
assert server_r["predicted_class"] == target

# Submit for the flag
result = requests.post(f"{BASE_URL}/submit",
                       json={"image_b64": b64_from_x01(x_adv)}, timeout=30).json()
print(result)
# {"success": true, "flag": "HTB{...}", "pixels_modified": X, ...}
```

---

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| No valid pixels early in attack | target gradient already negative for all pixels | Try pairwise JSMA (modify two pixels at once per step) |
| `L0 constraint violated` from server | Local and server counts differ | Server counts pixels with `|diff| > 1e-6`; ensure you're not creating float noise — use `np.clip((x*255).round()/255, 0, 1)` after each step |
| `Submission rejected: clean image` | Submitted the original `x` unchanged | Make sure you're submitting `x_adv`, not `x` |
| Local pred ≠ server pred | Model not loaded correctly, or normalisation applied wrong | Confirm `mnist_normalize` is applied inside forward pass only; inputs to ART/server are raw [0,1] |
| L2 constraint violated | Perturbed too many or too large pixels | Reduce `theta`, reduce `budget` or select pixels with lower magnitude change |

---

## Key Concepts Exercised

- **Jacobian computation** — gradient of each output class probability w.r.t. each input pixel
- **Saliency scoring** — jointly maximise target-class gradient and minimise sum of other gradients
- **L0 constraint** — pixel count, not magnitude — a single pixel changed massively still costs 1
- **Image space vs normalised space** — server operates in [0,1]; model normalises internally
- **Challenge API pattern** — GET /challenge → GET /weights → attack → POST /predict → POST /submit

---

## Related Pages

- [[attack/ai/jsma]] — full JSMA theory: saliency math, single-pixel vs pairwise, attack loop
- [[attack/ai/adversarial_examples]] — L0/L2/L∞ norm taxonomy and threat model
- [[attack/ai/elasticnet_attack]] — alternative sparse attack (L1+L2) using same API pattern
- [[tools/attack/art]] — ART library implementation of JSMA (`SaliencyMapMethod`)
- [[study_guide/coae]] — exam-day checklist and Python snippet reference

## Sources

- raw/ai_evasion_sparsity/skill_assessment.md
- raw/ai_evasion_sparsity/jsma_fundamentals.md
