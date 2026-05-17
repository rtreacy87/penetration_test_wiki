---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-14
source_count: 1
---

# Skills Assessment 2 — DeepFool Minimal Perturbation on CIFAR-10

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L2 (in normalized space)  
**Objective:** Find a minimal L2 perturbation that causes misclassification of a CIFAR-10 horse image (any wrong class counts).

## Scenario

A server exposes the same `CIFAR10CNN` from Skills Assessment 1. The challenge presents a horse image (class 7); you must cause any misclassification while keeping the **L2 distance in normalized space** ≤ 3.5. Unlike Skills Assessment 1 (targeted L∞), this uses DeepFool (untargeted L2) — the algorithm finds the closest decision boundary, takes the minimal step to cross it, and repeats.

**All three conditions must hold:**
- Predicted class differs from horse (class 7)
- `‖(x_adv − mean)/std − (x − mean)/std‖₂ ≤ 3.5` (L2 in normalized space)
- All pixel values remain in `[0, 1]³²×³²×³`

---

## Phase 0 — Environment Setup

```bash
mkdir sa2; cd sa2
python3 -m venv venv && source venv/bin/activate
pip3 install numpy requests torch torchvision pillow
```

---

## Phase 1 — API and Challenge Fetch

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
curl -s "$BASE_URL/challenge" | python3 -m json.tool
```

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Service check |
| `/challenge` | GET | Returns image, class, `l2_threshold`, `overshoot_hint`, `max_iterations_hint` |
| `/model/weights` | GET | Download `cifar10_model_best.pth` (~6.3 MB) |
| `/predict` | POST | Returns `{predicted_class, confidence}` |
| `/submit` | POST | Validate L2 (normalized) + misclassification; returns flag |

```python
import os, io, base64, requests
import torch
import torch.nn as nn
import torch.nn.functional as F
import torchvision.transforms as transforms
from PIL import Image
import numpy as np

BASE_URL = os.getenv("BASE_URL", "http://127.0.0.1:8001")

def base64_to_tensor(b64: str) -> torch.Tensor:
    img = Image.open(io.BytesIO(base64.b64decode(b64)))
    return transforms.ToTensor()(img)          # (3, 32, 32) in [0,1]

def tensor_to_base64(tensor: torch.Tensor) -> str:
    arr = (tensor.permute(1, 2, 0).numpy() * 255).astype(np.uint8)
    buf = io.BytesIO()
    Image.fromarray(arr).save(buf, format="PNG")
    buf.seek(0)
    return base64.b64encode(buf.getvalue()).decode("utf-8")

def l2_normalized(a: torch.Tensor, b: torch.Tensor,
                  mean: list, std: list) -> float:
    """L2 distance between two [0,1] images in normalized space."""
    mean_t = torch.tensor(mean).view(3, 1, 1)
    std_t  = torch.tensor(std).view(3, 1, 1)
    return float(torch.norm((a - mean_t) / std_t - (b - mean_t) / std_t))

ch           = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x            = base64_to_tensor(ch["image"])           # (3, 32, 32) in [0,1]
orig_class   = int(ch["original_class"])               # 7 (horse)
l2_threshold = float(ch["l2_threshold"])               # 3.5
overshoot    = float(ch["overshoot_hint"])             # 0.02
max_iter     = int(ch["max_iterations_hint"])          # 50
mean         = ch["normalization"]["mean"]
std          = ch["normalization"]["std"]
print(f"orig={orig_class}  l2_threshold={l2_threshold}  overshoot={overshoot}")
```

---

## Phase 2 — Model Architecture and Weight Loading

Same `CIFAR10CNN` as Skills Assessment 1 — raw logits output, external normalization:

```python
class CIFAR10CNN(nn.Module):
    def __init__(self, num_classes: int = 10):
        super().__init__()
        self.conv1   = nn.Conv2d(3, 32, kernel_size=3, padding=1)
        self.bn1     = nn.BatchNorm2d(32)
        self.relu1   = nn.ReLU()
        self.pool1   = nn.MaxPool2d(2, 2)
        self.conv2   = nn.Conv2d(32, 64, kernel_size=3, padding=1)
        self.bn2     = nn.BatchNorm2d(64)
        self.relu2   = nn.ReLU()
        self.pool2   = nn.MaxPool2d(2, 2)
        self.fc1     = nn.Linear(64 * 8 * 8, 128)
        self.relu3   = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        self.fc2     = nn.Linear(128, num_classes)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.pool1(self.relu1(self.bn1(self.conv1(x))))
        x = self.pool2(self.relu2(self.bn2(self.conv2(x))))
        x = x.view(x.size(0), -1)
        x = self.dropout(self.relu3(self.fc1(x)))
        return self.fc2(x)                     # raw logits

def load_model(path: str, device: str = "cpu") -> CIFAR10CNN:
    model = CIFAR10CNN()
    ckpt  = torch.load(path, map_location=device)
    if isinstance(ckpt, dict) and "model_state_dict" in ckpt:
        model.load_state_dict(ckpt["model_state_dict"])
    else:
        model.load_state_dict(ckpt)
    return model.to(device).eval()

weights_path = "cifar10_model_best.pth"
if not os.path.exists(weights_path):
    resp = requests.get(f"{BASE_URL}/model/weights", timeout=30)
    resp.raise_for_status()
    with open(weights_path, "wb") as f:
        f.write(resp.content)

device = "cuda" if torch.cuda.is_available() else "cpu"
model  = load_model(weights_path, device)

mean_t = torch.tensor(mean).view(3, 1, 1)
std_t  = torch.tensor(std).view(3, 1, 1)
with torch.no_grad():
    orig_pred = model(((x - mean_t) / std_t).unsqueeze(0).to(device)).argmax(dim=1).item()
print(f"Clean prediction: {orig_pred}  (expected: {orig_class})")
```

---

## Phase 3 — DeepFool Untargeted Attack

### Algorithm overview

DeepFool linearizes the classifier at the current point and finds the closest decision boundary across all K−1 competitor classes. It steps directly to that boundary (plus a small overshoot), then re-linearizes and repeats until misclassification.

At each iteration:
1. For every class `k ≠ current_class`, compute:
   - `w_k = ∇logit[k] − ∇logit[current]` (boundary normal in normalized space)
   - `f_k = logit[k] − logit[current]` (signed distance to boundary)
   - `dist_k = |f_k| / ‖w_k‖` (L2 distance to the linearized boundary)
2. Select the class `k*` with minimum `dist_k`.
3. Compute the minimal step: `r_i = (|f_{k*}| / ‖w_{k*}‖²) · w_{k*}` (in normalized space)
4. Convert to pixel space: `r_i_pixel = r_i · std` (chain rule)
5. Accumulate: `r_total += (1 + overshoot) · r_i_pixel`

The perturbation is accumulated in **pixel space**. The server measures L2 in **normalized space** — so `‖r_total / std‖₂` is what must stay below `l2_threshold`.

```python
def deepfool_attack(
    model: nn.Module,
    image: torch.Tensor,
    mean: list,
    std: list,
    num_classes: int = 10,
    overshoot: float = 0.02,
    max_iter: int = 50,
    device: str = "cpu",
) -> tuple:
    """
    DeepFool untargeted attack on [0,1] CIFAR-10 image.

    Returns (adversarial_image, perturbation, iterations, final_class)
    L2 constraint is measured in normalized space (divide by std before norm).
    """
    mean_t = torch.tensor(mean, device=device).view(3, 1, 1)
    std_t  = torch.tensor(std,  device=device).view(3, 1, 1)

    x      = image.clone().to(device)
    x_orig = image.clone().to(device)
    r_total = torch.zeros_like(x)

    x_norm  = (x - mean_t) / std_t
    with torch.no_grad():
        logits      = model(x_norm.unsqueeze(0))
        current_cls = logits.argmax(dim=1).item()
    original_cls = current_cls

    for iteration in range(max_iter):
        x_norm = (x - mean_t) / std_t
        x_norm.requires_grad = True
        logits      = model(x_norm.unsqueeze(0))
        current_cls = logits.argmax(dim=1).item()

        if current_cls != original_cls:
            print(f"  Misclassified at iteration {iteration+1}: {current_cls}")
            break

        # Find closest decision boundary across all other classes
        min_dist = float("inf")
        best_w, best_f = None, None

        for k in range(num_classes):
            if k == current_cls:
                continue

            if x_norm.grad is not None:
                x_norm.grad.zero_()
            logits[0, k].backward(retain_graph=True)
            grad_k = x_norm.grad.clone()

            if x_norm.grad is not None:
                x_norm.grad.zero_()
            logits[0, current_cls].backward(retain_graph=True)
            grad_curr = x_norm.grad.clone()

            w_k   = grad_k - grad_curr                       # boundary normal (normalized space)
            f_k   = (logits[0, k] - logits[0, current_cls]).item()
            dist  = abs(f_k) / (torch.norm(w_k) + 1e-10)

            if dist < min_dist:
                min_dist = dist
                best_w, best_f = w_k, f_k

        # Minimal step in normalized space
        w_norm_sq = torch.norm(best_w) ** 2
        r_i       = (abs(best_f) / (w_norm_sq + 1e-10)) * best_w

        # Convert to pixel space: chain rule through x_norm = (x - mean) / std
        r_i_pixel  = r_i * std_t

        r_total = r_total + (1 + overshoot) * r_i_pixel
        x = torch.clamp(x_orig + r_total, 0.0, 1.0)

        if (iteration + 1) % 10 == 0:
            with torch.no_grad():
                x_n   = (x - mean_t) / std_t
                xo_n  = (x_orig - mean_t) / std_t
                l2    = torch.norm(x_n - xo_n).item()
            print(f"  Iter {iteration+1}/{max_iter}  L2_norm={l2:.4f}")

    return x.detach().cpu(), r_total.detach().cpu(), iteration + 1, current_cls
```

Run the attack:

```python
adv_image, _, iters, final_cls = deepfool_attack(
    model, x, mean, std,
    num_classes=ch["num_classes_hint"],
    overshoot=ch["overshoot_hint"],
    max_iter=ch["max_iterations_hint"],
    device=device,
)
print(f"Iterations: {iters}  Final class: {final_cls}")

l2 = l2_normalized(adv_image, x, mean, std)
print(f"L2 (normalized): {l2:.4f}  (threshold: {l2_threshold})")
print(f"Within threshold: {l2 <= l2_threshold}")
```

---

## Phase 4 — Validate and Submit

```bash
# Quick CLI prediction check
curl -s -X POST "$BASE_URL/predict" \
  -H "content-type: application/json" \
  -d "{\"image\": \"<base64_adv_image>\"}" | python3 -m json.tool
```

```python
adv_b64 = tensor_to_base64(adv_image)
resp = requests.post(f"{BASE_URL}/submit", json={"image": adv_b64}, timeout=15)
resp.raise_for_status()
result = resp.json()

val = result["validation"]
print(f"L2 norm:        {val['l2_norm']:.4f} / {val['l2_threshold']}")
print(f"L2 satisfied:   {val['l2_satisfied']}")
print(f"Valid range:    {val['valid_range']}")
print(f"Original class: {val['original_class']}")
print(f"Adv class:      {val['adversarial_class']}")
print(f"Misclassified:  {val['misclassification']}")
print(f"Success:        {result['success']}")
if result["success"]:
    print(f"Flag: {result['flag']}")
```

Expected output:
```
L2 norm: 0.9643 / 3.5
L2 satisfied: True
Valid range: True
Original prediction: horse
Adversarial prediction: truck
Misclassification: True
```

Validation errors:

| Server response | Cause | Fix |
|----------------|-------|-----|
| `l2_satisfied: false` | L2 exceeds threshold | Decrease `overshoot`; use fewer iterations |
| `misclassification: false` | Attack converged too early | Increase `max_iter` or `overshoot` |
| HTTP 400 invalid PNG | Wrong field or encoding | Use `"image"` key, not `"image_b64"` |

---

## Lessons Learned

- **L2 in normalized space**: The server measures `‖(x_adv−mean)/std − (x−mean)/std‖₂ = ‖(x_adv−x)/std‖₂`. This is usually much smaller than the pixel-space L2. A threshold of 3.5 in normalized space corresponds to roughly 3.5 × 0.25 ≈ 0.875 in pixel space per channel — relatively generous.
- **Perturbation space conversion**: DeepFool computes `r_i` in normalized space (from `w_k = ∇logit[k] − ∇logit[current]` with gradients taken on `x_norm`). Multiplying by `std_t` converts to pixel space via chain rule: `∂x_norm/∂x = 1/std`, so `∂L/∂x = ∂L/∂x_norm / std`, and inversely a perturbation vector in normalized space scales to pixel space by `std`.
- **DeepFool vs. I-FGSM**: DeepFool finds the minimum perturbation to the nearest decision boundary. I-FGSM optimizes loss per iteration without this geometric insight — it is effective for L∞ budgets but less efficient for L2. The threshold of 3.5 (normalized) is generous enough that either approach works; DeepFool typically converges in ≤ 5 iterations here.
- **Overshoot accounts for PNG rounding**: 8-bit quantization when saving the PNG can push the image back across the boundary. An overshoot of 0.02–0.05 is usually enough for CIFAR-10 at this threshold.
- **No targeted class needed**: Any misclassification satisfies the challenge. The model often jumps to the semantically nearest class (horse → truck, deer, or automobile are common).

---

## Related Pages

- [[attack/ai/deepfool]] — full DeepFool theory: boundary linearization, ρ_adv robustness metric
- [[attack/ai/i_fgsm]] — iterative L∞ attack; contrast with this L2 approach
- [[attack/ai/adversarial_examples]] — norm taxonomy: L0/L1/L2/L∞ and why L2 favors minimal diffuse noise
- [[labs/htb/ai_evasion_first_order_attacks/skills_assessment_1]] — same model, targeted I-FGSM L∞
- [[labs/htb/ai_evasion_first_order_attacks/deepfool_challenge]] — MNIST targeted DeepFool in pixel L2

## Sources

- raw/lab/ai_evasion_first_order_attacks/skills_assessemnt_2.md
