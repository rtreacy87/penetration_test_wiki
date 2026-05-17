---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-14
source_count: 1
---

# FGSM Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L∞  
**Objective:** Craft an adversarial MNIST image that causes misclassification while keeping the L∞ pixel-space perturbation ≤ ε.

## Scenario

A server exposes a fixed MNIST digit classifier. You receive a baseline image, its true label, and an epsilon budget. Download the model weights, compute the gradient locally, apply a single FGSM step in pixel space, and submit. The server validates `‖x_adv − x‖∞ ≤ ε` and that the model misclassifies before returning the flag.

---

## Phase 0 — Environment Setup

The HTB Academy workstation may not have enough disk space to install PyTorch. Free space first:

```bash
sudo apt autoremove
sudo apt clean
sudo rm -rf /tmp/*
sudo apt purge seclists burpsuite ghidra wireshark powershell-empire bloodhound exploitdb metasploit-framework dotnet neo4j zaproxy
sudo rm -rf /usr/share/icons/Hack-The-Box-Icons
sudo tune2fs -m1 /dev/vda1
pip install torch --index-url https://download.pytorch.org/whl/cpu
```

If running on your own machine, use a virtualenv instead:

```bash
mkdir fgsm_challenge; cd fgsm_challenge
python3 -m venv venv && source venv/bin/activate
pip3 install torch torchvision numpy requests pillow
```

---

## Phase 1 — API Setup and Challenge Fetch

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
# {"status":"ok","epsilon":0.25,"index":2}
```

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Service check; returns `epsilon` and sample `index` |
| `/challenge` | GET | Returns `{sample_index, label, epsilon, image_b64}` |
| `/predict` | POST | Sanity check; returns `{pred, confidence}` |
| `/weights` | GET | Download model `state_dict` (binary `.pth`) |
| `/submit` | POST | Validate L∞ + misclassification; returns flag |

All images are base64-encoded 28×28 grayscale PNGs. Pixel values are **raw [0,1]** — not pre-normalized tensors.

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
from PIL import Image

BASE_URL = os.getenv("BASE_URL", "http://127.0.0.1:8000")
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

# Fetch challenge
ch = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x   = x01_from_b64_png(ch["image_b64"])   # (28,28) baseline in [0,1]
label = int(ch["label"])
eps   = float(ch["epsilon"])
print(f"label={label}  epsilon={eps}")
```

---

## Phase 2 — Model Architecture and Weight Loading

The server uses `SimpleClassifier`. The model does **not** normalize internally; callers pass already-normalized tensors via the separate `mnist_normalize()` helper. Keep these two functions separate so your gradients compute correctly.

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
        # Input: already-normalized tensor (callers use mnist_normalize)
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

# Download weights and load
wt = requests.get(f"{BASE_URL}/weights", timeout=15).content
with open("fgsm_weights.pth", "wb") as f:
    f.write(wt)
model = SimpleClassifier()
model.load_state_dict(torch.load("fgsm_weights.pth", map_location="cpu"))
model.eval()

# Sanity: local prediction should match the challenge label
x_t = torch.from_numpy(x[None, None, ...]).float()
local_pred = int(torch.argmax(model(mnist_normalize(x_t)), dim=1).item())
print(f"Local clean pred: {local_pred}  (expected: {label})")
```

---

## Phase 3 — FGSM Attack in Pixel Space

FGSM computes the sign of the loss gradient with respect to the **pixel-space** input and shifts the image by `ε` in that direction. Because normalization happens outside the model (via `mnist_normalize`), autograd traces through it and delivers the gradient already in pixel space — no manual chain-rule division required.

```python
def fgsm_untargeted(model, x01: np.ndarray, y: int, epsilon: float) -> np.ndarray:
    """
    Craft FGSM adversarial example under L_inf in [0,1] pixel space.

    The pixel-space input x has requires_grad=True. PyTorch traces through
    mnist_normalize automatically, so x.grad is already ∂L/∂x in pixel space.
    No need to divide by MNIST_STD manually.
    """
    x = torch.from_numpy(x01).float().requires_grad_(True)
    target = torch.tensor([y], dtype=torch.long)

    logits = model(mnist_normalize(x))
    loss = nn.NLLLoss()(logits, target)
    model.zero_grad()
    loss.backward()

    grad_sign = x.grad.detach().sign()
    x_adv = x.detach() + epsilon * grad_sign
    return torch.clamp(x_adv, 0.0, 1.0).numpy()
```

### Handling PNG quantization

When saving to 8-bit PNG, pixel values are rounded to the nearest integer and re-scaled to [0,1]. This can push `‖x_adv − x‖∞` above `epsilon` by up to `1/255 ≈ 0.004`. Use a small margin to account for this:

```python
# Try conservative epsilons; fall back to 99% epsilon if all fail
for margin in [1.0/255.0 + 1e-4, 2e-3, 1e-3, 5e-4, 0.0]:
    eps_eff = max(0.0, eps - margin)
    cand = fgsm_untargeted(model, x[None, None, ...].astype(np.float32), label, eps_eff)
    cand_pred = int(torch.argmax(
        model(mnist_normalize(torch.from_numpy(cand))), dim=1).item())
    if cand_pred != label:
        x_adv = cand
        break
else:
    x_adv = fgsm_untargeted(model, x[None, None, ...].astype(np.float32), label, eps * 0.99)
```

---

## Phase 4 — Validate and Submit

```python
x2d  = x_adv[0, 0]   # collapse batch and channel dims: (28,28)
linf = float(np.max(np.abs(x2d - x)))
print(f"L∞ = {linf:.4f}  (budget: {eps})")   # must be ≤ eps

b64 = b64_png_from_x01(x2d)
resp = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64}, timeout=10)
resp.raise_for_status()
print(resp.json())
# {"ok": true, "pred": 8, "linf": 0.246, "flag": "HTB{...}"}
```

Validation errors and their meanings:

| HTTP status | Message | Fix |
|-------------|---------|-----|
| 400 | `"L_inf too large: 0.26 > 0.25"` | Subtract a larger margin from `epsilon` |
| 400 | `"Misclassification not achieved"` | Model still predicts the true label — increase `epsilon` or use I-FGSM |
| 400 | `"Image must be 28x28"` | Wrong image dimensions in base64 payload |

---

## Lessons Learned

- **Normalization is external**: `SimpleClassifier.forward()` takes already-normalized tensors. Callers must apply `mnist_normalize()`. Embedding normalization inside the forward method (a common tutorial shortcut) would change the architecture and break weight compatibility.
- **Gradient space is pixel space**: Because `x` (pixel space) has `requires_grad=True` and autograd traces through `mnist_normalize(x)`, `x.grad` is already `∂L/∂x` in pixel space. No manual `/ MNIST_STD` is needed.
- **PNG quantization adds ≤ 1/255**: Use `eps - 1/255` as the effective budget, or implement a retry loop with decreasing margins.
- **Single-step failure**: FGSM may not always misclassify at the given ε. If the prediction is unchanged, try [[attack/ai/i_fgsm]] with 10–20 iterations at the same budget.

---

## Related Pages

- [[attack/ai/fgsm]] — full FGSM theory: gradient sign formulation, epsilon budgets, targeted variant
- [[attack/ai/i_fgsm]] — iterative version; higher success rate at same epsilon
- [[attack/ai/adversarial_examples]] — L∞/L2/L0 norm taxonomy and threat model
- [[labs/htb/ai_evasion_first_order_attacks/deepfool_challenge]] — L2 targeted variant
- [[labs/htb/ai_evasion_first_order_attacks/skills_assessment_1]] — CIFAR-10 I-FGSM targeted

## Sources

- raw/lab/ai_evasion_first_order_attacks/FGSM_challenge.md
