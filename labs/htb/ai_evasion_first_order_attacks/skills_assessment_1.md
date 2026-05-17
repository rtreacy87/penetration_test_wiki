---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-14
source_count: 1
---

# Skills Assessment 1 — Targeted I-FGSM on CIFAR-10

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L∞  
**Objective:** Transform a CIFAR-10 dog image into one the classifier predicts as cat while keeping `‖x_adv − x‖∞ ≤ 8/255`.

## Scenario

A server exposes a CIFAR-10 classifier trained on 32×32 RGB images. The challenge is fixed: a dog (class 5) must be misclassified as cat (class 3). The L∞ budget is `ε = 8/255 ≈ 0.0314`. Unlike the MNIST challenges, the model returns **raw logits** (not log-softmax), uses BatchNorm, and expects per-channel normalization with CIFAR-10 statistics.

**All three conditions must hold to receive the flag:**
- Predicted class equals cat (class 3)
- `‖x_adv − x‖∞ ≤ 8/255` in pixel space
- All pixel values remain in `[0, 1]³²×³²×³`

---

## Phase 0 — Environment Setup

```bash
mkdir sa1; cd sa1
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
| `/challenge` | GET | Returns image, original/target class, epsilon, normalization params |
| `/model/weights` | GET | Download `cifar10_model_best.pth` (~6.3 MB) |
| `/predict` | POST | Validate pipeline; returns `{predicted_class, confidence}` |
| `/submit` | POST | Validate L∞ + target; returns flag |

Images are base64-encoded 32×32 **RGB** PNGs (`image` field, not `image_b64`).

```python
import os, io, base64, requests
import torch
import torch.nn as nn
import torch.nn.functional as F
import torchvision.transforms as transforms
from PIL import Image
import numpy as np

BASE_URL = os.getenv("BASE_URL", "http://127.0.0.1:8000")

def base64_to_tensor(b64: str) -> torch.Tensor:
    """Decode base64 PNG to (C, H, W) float tensor in [0,1]."""
    img = Image.open(io.BytesIO(base64.b64decode(b64)))
    return transforms.ToTensor()(img)          # (3, 32, 32) in [0,1]

def tensor_to_base64(tensor: torch.Tensor) -> str:
    """Encode (C, H, W) float tensor in [0,1] to base64 PNG."""
    arr = (tensor.permute(1, 2, 0).numpy() * 255).astype(np.uint8)
    img = Image.fromarray(arr)
    buf = io.BytesIO()
    img.save(buf, format="PNG")
    buf.seek(0)
    return base64.b64encode(buf.getvalue()).decode("utf-8")

# Fetch challenge
ch          = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x           = base64_to_tensor(ch["image"])         # (3, 32, 32) in [0,1]
orig_class  = int(ch["original_class"])             # 5 (dog)
target_class = int(ch["target_class"])              # 3 (cat)
epsilon     = float(ch["epsilon"])                  # 0.031373 = 8/255
mean        = ch["normalization"]["mean"]           # [0.4914, 0.4822, 0.4465]
std         = ch["normalization"]["std"]            # [0.247, 0.2435, 0.2616]
print(f"orig={orig_class}  target={target_class}  epsilon={epsilon:.6f}")
```

---

## Phase 2 — Model Architecture and Weight Loading

`CIFAR10CNN` is a two-block convolutional network that returns **raw logits**. Use `F.cross_entropy` (not `nll_loss`) for the loss — it applies softmax internally.

```python
class CIFAR10CNN(nn.Module):
    def __init__(self, num_classes: int = 10):
        super().__init__()
        self.conv1  = nn.Conv2d(3, 32, kernel_size=3, padding=1)
        self.bn1    = nn.BatchNorm2d(32)
        self.relu1  = nn.ReLU()
        self.pool1  = nn.MaxPool2d(2, 2)       # 32×32 → 16×16
        self.conv2  = nn.Conv2d(32, 64, kernel_size=3, padding=1)
        self.bn2    = nn.BatchNorm2d(64)
        self.relu2  = nn.ReLU()
        self.pool2  = nn.MaxPool2d(2, 2)       # 16×16 → 8×8
        self.fc1    = nn.Linear(64 * 8 * 8, 128)
        self.relu3  = nn.ReLU()
        self.dropout = nn.Dropout(0.5)
        self.fc2    = nn.Linear(128, num_classes)

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

# Download weights if missing
weights_path = "cifar10_model_best.pth"
if not os.path.exists(weights_path):
    resp = requests.get(f"{BASE_URL}/model/weights", timeout=30)
    resp.raise_for_status()
    with open(weights_path, "wb") as f:
        f.write(resp.content)
    print(f"Weights saved to {weights_path}")

device = "cuda" if torch.cuda.is_available() else "cpu"
model  = load_model(weights_path, device)

# Verify original prediction
mean_t = torch.tensor(mean).view(3, 1, 1)
std_t  = torch.tensor(std).view(3, 1, 1)
with torch.no_grad():
    x_norm = (x - mean_t) / std_t
    pred   = model(x_norm.unsqueeze(0).to(device)).argmax(dim=1).item()
print(f"Verified clean prediction: {pred}  (expected: {orig_class})")
```

---

## Phase 3 — I-FGSM Targeted Attack

I-FGSM applies many small FGSM steps. For a **targeted** attack, we minimize the cross-entropy loss toward the target class, so each update moves in the **negative** gradient direction.

### Gradient space conversion

The model does not normalize internally. The caller normalizes: `x_norm = (x_adv − mean) / std`. Setting `requires_grad=True` on `x_norm` gives `∂L/∂x_norm` after backward. Converting to pixel-space gradient via the chain rule:

```
∂L/∂x_adv = ∂L/∂x_norm · ∂x_norm/∂x_adv = grad_norm / std
```

Take the sign of `grad_pixel` and step in the negative direction (toward lower loss = toward target class):

```python
def ifgsm_targeted(
    model: nn.Module,
    image: torch.Tensor,
    target_class: int,
    epsilon: float,
    mean: list,
    std: list,
    num_iterations: int = 50,
    alpha: float = None,
    device: str = "cpu",
) -> torch.Tensor:
    """
    Iterative targeted FGSM.

    alpha defaults to epsilon / num_iterations for fine-grained control.
    Targeted attack: step in NEGATIVE gradient direction (minimize loss toward target).
    """
    if alpha is None:
        alpha = epsilon / num_iterations

    mean_t = torch.tensor(mean, device=device).view(3, 1, 1)
    std_t  = torch.tensor(std,  device=device).view(3, 1, 1)
    x_orig = image.clone().to(device)
    x_adv  = image.clone().to(device)
    target = torch.tensor([target_class], device=device)

    for i in range(num_iterations):
        # Normalize for model input; track gradients at the normalized layer
        x_norm = (x_adv - mean_t) / std_t
        x_norm.requires_grad = True

        outputs = model(x_norm.unsqueeze(0))
        loss    = F.cross_entropy(outputs, target)
        model.zero_grad()
        loss.backward()

        # Chain rule: convert gradient from normalized space to pixel space
        grad_pixel = x_norm.grad / std_t

        # Targeted: NEGATIVE gradient step (descent toward target class)
        x_adv = x_adv - alpha * grad_pixel.sign()

        # Project perturbation back onto L∞ ball around original image
        delta = torch.clamp(x_adv - x_orig, -epsilon, epsilon)
        x_adv = torch.clamp(x_orig + delta, 0.0, 1.0).detach()

        if (i + 1) % 10 == 0:
            with torch.no_grad():
                pred = model((x_adv - mean_t) / std_t).argmax(dim=1).item()
            print(f"  Iter {i+1}/{num_iterations}: pred={pred}")

    return x_adv.cpu()
```

Run the attack:

```python
adv_image = ifgsm_targeted(
    model, x, target_class, epsilon, mean, std,
    num_iterations=50, device=device
)

# Verify locally
with torch.no_grad():
    adv_norm = (adv_image - mean_t) / std_t
    adv_pred = model(adv_norm.unsqueeze(0).to(device)).argmax(dim=1).item()
linf = float(torch.abs(adv_image - x).max())
print(f"Adversarial pred: {adv_pred}  L∞={linf:.6f}  (budget: {epsilon:.6f})")
```

---

## Phase 4 — Validate and Submit

```bash
# Quick CLI check before submitting
curl -s -X POST "$BASE_URL/predict" \
  -H "content-type: application/json" \
  -d "{\"image\": \"$(python3 -c 'import base64,sys; print(open(\"adv.png\",\"rb\").read())' | base64 | tr -d '\n')\"}" | python3 -m json.tool
```

```python
adv_b64 = tensor_to_base64(adv_image)
resp = requests.post(f"{BASE_URL}/submit", json={"image": adv_b64}, timeout=15)
resp.raise_for_status()
result = resp.json()

print(f"L∞ satisfied: {result['validation']['linf_satisfied']}")
print(f"Valid range:  {result['validation']['valid_range']}")
print(f"Adversarial:  {result['validation']['adversarial_class']}")
print(f"Target:       {result['validation']['target_class']}")
print(f"Success:      {result['success']}")
if result['success']:
    print(f"Flag: {result['flag']}")
```

Expected output:
```
L∞ norm: 0.031373 / 0.031373
L∞ constraint satisfied: True
Valid range [0,1]: True
Adversarial prediction: cat
Target achieved: True
```

Validation errors:

| HTTP status | Message | Fix |
|-------------|---------|-----|
| `success=false` | `"Target not achieved"` | Increase iterations (100) or reduce alpha |
| `success=false` | `"linf_satisfied: false"` | Clip delta to `[-epsilon, epsilon]` before adding back |
| 400 | `"invalid base64"` | Use the `"image"` field, not `"image_b64"` |

---

## Lessons Learned

- **Raw logits + cross_entropy**: `CIFAR10CNN` returns raw logits (no softmax). Use `F.cross_entropy` (which applies softmax internally), not `nll_loss` (which requires log-probabilities).
- **Chain rule for gradient conversion**: Setting `requires_grad=True` on `x_norm` gives `∂L/∂x_norm`. Divide by `std_t` to convert to `∂L/∂x_adv` in pixel space. (For MNIST with external normalization, you can alternatively set `requires_grad=True` on `x` directly and let autograd trace through normalization.)
- **Targeted = negative step**: For a targeted attack, you *minimize* the loss toward the target class, so the gradient sign is negated compared to an untargeted attack.
- **L∞ projection after every step**: The `torch.clamp(delta, -epsilon, epsilon)` step is critical. Without it, individual iterations can accumulate unlimited perturbation.
- **No GPU required**: CPU inference on CIFAR-10 with 50 iterations completes in under 30 seconds.

---

## Related Pages

- [[attack/ai/i_fgsm]] — I-FGSM/BIM/PGD theory: iterative L∞ with projection
- [[attack/ai/fgsm]] — single-step variant; simpler but lower success rate
- [[attack/ai/adversarial_examples]] — threat model: white-box vs black-box, norm taxonomy
- [[labs/htb/ai_evasion_first_order_attacks/skills_assessment_2]] — same model, DeepFool untargeted L2 attack
- [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]] — MNIST baseline challenge

## Sources

- raw/lab/ai_evasion_first_order_attacks/skills_assessemnt_1.md
