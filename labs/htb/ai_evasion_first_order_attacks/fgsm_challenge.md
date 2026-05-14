---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-13
source_count: 1
---

# FGSM Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L∞  
**Objective:** Craft an adversarial MNIST image that causes misclassification while staying within ε in L∞ pixel space.

## Overview

The challenge exposes a deployed MNIST classifier through an HTTP API. The target image is fixed (deterministic); you download the model weights, compute gradients locally, craft an adversarial example with FGSM, and submit. The server validates both the L∞ constraint and misclassification before returning the flag.

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Returns `{status, epsilon, index}` |
| `/challenge` | GET | Returns `{sample_index, label, epsilon, image_b64}` |
| `/predict` | POST | Test pipeline; returns `{pred, confidence}` |
| `/weights` | GET | Download model `state_dict` (binary) |
| `/submit` | POST | Validate and retrieve flag |

All images are base64-encoded 28×28 grayscale PNGs in `[0,1]` pixel space (not normalized tensors).

## Environment Setup

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | jq
# {"status":"ok","epsilon":0.25,"index":2}
```

```python
import os, io, base64, numpy as np, requests, torch, torch.nn as nn
from PIL import Image

BASE_URL = os.getenv("BASE_URL", "http://127.0.0.1:8000")
MNIST_MEAN, MNIST_STD = 0.1307, 0.3081

def x01_from_b64_png(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0.0, 1.0)

def b64_png_from_x01(x2d):
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img = Image.fromarray(x255, mode="L")
    buf = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")
```

## Model Architecture

The server uses `SimpleClassifier` — normalizes inputs internally, then two conv layers + two FC layers, log-softmax output:

```python
class SimpleClassifier(nn.Module):
    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(1, 32, 3, 1)
        self.conv2 = nn.Conv2d(32, 64, 3, 1)
        self.dropout1 = nn.Dropout(0.25)
        self.dropout2 = nn.Dropout(0.5)
        self.fc1 = nn.Linear(9216, 128)
        self.fc2 = nn.Linear(128, 10)

    def forward(self, x01):   # expects [0,1] input, normalizes internally
        x = (x01 - MNIST_MEAN) / MNIST_STD
        x = torch.relu(self.conv1(x))
        x = torch.relu(self.conv2(x))
        x = torch.max_pool2d(x, 2)
        x = self.dropout1(x)
        x = torch.flatten(x, 1)
        x = torch.relu(self.fc1(x))
        x = self.dropout2(x)
        return torch.log_softmax(self.fc2(x), dim=1)
```

## Attack Steps

### 1. Fetch challenge and load weights

```python
ch = requests.get(f"{BASE_URL}/challenge", timeout=10).json()
x = x01_from_b64_png(ch["image_b64"])    # (28,28) float32 in [0,1]
label = int(ch["label"])
eps = float(ch["epsilon"])               # pixel-space L∞ budget

wt = requests.get(f"{BASE_URL}/weights", timeout=10).content
open("fgsm_weights.pth", "wb").write(wt)
model = SimpleClassifier().eval()
model.load_state_dict(torch.load("fgsm_weights.pth", map_location="cpu"))
```

### 2. Verify local model matches server

```python
x_t = torch.from_numpy(x[None, None, ...]).float()
local_pred = int(torch.argmax(model(x_t), dim=1).item())
server_pred = requests.post(f"{BASE_URL}/predict",
    json={"image_b64": b64_png_from_x01(x)}).json()["pred"]
assert local_pred == server_pred
```

### 3. Compute FGSM perturbation in pixel space

The model normalizes inputs internally, so gradients are computed in normalized space and must be converted back to pixel space before applying the pixel-space ε budget:

```python
x_t = torch.from_numpy(x[None, None, ...]).float().requires_grad_(True)
loss = torch.nn.functional.nll_loss(model(x_t),
                                     torch.tensor([label]))
loss.backward()
# Convert gradient from normalized space to pixel space
grad_pixel = x_t.grad.data.squeeze() / MNIST_STD
x_adv = np.clip(x + eps * np.sign(grad_pixel.numpy()), 0.0, 1.0)
```

### 4. Verify constraint and submit

```python
linf = float(np.max(np.abs(x_adv - x)))
print(f"L∞ = {linf:.4f} (budget: {eps})")   # must be ≤ eps

resp = requests.post(f"{BASE_URL}/submit",
    json={"image_b64": b64_png_from_x01(x_adv)}).json()
print(resp)   # {"ok": true, "pred": ..., "linf": ..., "flag": "HTB{...}"}
```

## Lessons Learned

- The model normalizes inputs internally (`x_norm = (x - μ) / σ`). Gradients flow back through this normalization, so they live in normalized space. Dividing by σ before applying the pixel-space ε is required — without the conversion, the effective perturbation is `ε · σ` in pixel space, which typically exceeds the budget.
- The API rejects submissions with `linf > epsilon` (HTTP 400) with a message like `"L_inf too large: 0.26 > 0.25"`. Test with the clean image first to confirm the pipeline before submitting adversarials.
- Single-step FGSM may not always succeed at the given ε. If the attack fails (prediction unchanged), try [[attack/ai/i_fgsm]] with 10–20 iterations at the same budget.

## Related Pages

- [[attack/ai/fgsm]]
- [[attack/ai/i_fgsm]]
- [[attack/ai/adversarial_examples]]

## Sources

- raw/ai_evasion_first_order_attacks/fgsm_challenge.md
