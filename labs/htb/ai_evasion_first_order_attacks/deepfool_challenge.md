---
tags: [lab, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-13
source_count: 1
---

# DeepFool Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion — First-Order Attacks  
**Constraint type:** L2  
**Objective:** Craft a *targeted* adversarial example — the model must predict a specific target class, and the L2 distance from the clean image must stay within the server's threshold.

## Overview

Unlike the [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]] (which uses L∞ and accepts any misclassification), this challenge requires:
1. The adversarial image is classified as a **specific target class** (not just any wrong class).
2. The L2 distance from the clean baseline is within `l2_threshold`.

Use a DeepFool-style iterative targeted attack: at each iteration, steer the gradient toward the target class, then check if the prediction has flipped. Because the L2 constraint rewards minimal perturbations, DeepFool's geometry-aware steps are well-suited here.

## API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Returns `{status, l2_threshold, index, target}` |
| `/challenge` | GET | Returns `{sample_index, label, target, l2_threshold, image_b64}` |
| `/predict` | POST | Test prediction; returns `{pred, confidence}` |
| `/weights` | GET | Download model `state_dict` |
| `/submit` | POST | Validate and retrieve flag |

## Environment Setup

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | jq
# {"status":"ok","l2_threshold":0.75,"index":95,"target":6}
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

def l2(a, b):
    return float(np.linalg.norm((a - b).ravel(), ord=2))
```

## Model Architecture

Same `SimpleClassifier` as the FGSM challenge — normalizes inputs internally, log-softmax output:

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

    def forward(self, x01):
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
x = x01_from_b64_png(ch["image_b64"])     # (28,28) float32 in [0,1]
label = int(ch["label"])
target = int(ch["target"])
threshold = float(ch["l2_threshold"])

wt = requests.get(f"{BASE_URL}/weights", timeout=10).content
open("deepfool_weights.pth", "wb").write(wt)
model = SimpleClassifier().eval()
model.load_state_dict(torch.load("deepfool_weights.pth", map_location="cpu"))
```

### 2. Targeted iterative attack

Use a targeted I-FGSM-style approach: minimize loss toward the target class at each step, then project onto an L2 ball to keep the perturbation bounded:

```python
def targeted_l2_attack(model, x_np, target, threshold,
                        num_iter=200, step_size=0.005):
    """Targeted iterative attack with L2 projection."""
    x_orig = torch.from_numpy(x_np[None, None, ...]).float()
    x_adv = x_orig.clone()
    target_t = torch.tensor([target])

    for _ in range(num_iter):
        x_adv = x_adv.detach().requires_grad_(True)
        loss = torch.nn.functional.nll_loss(model(x_adv), target_t)
        loss.backward()

        # Gradient descent toward target (minimize loss for target class)
        x_adv = x_adv - step_size * x_adv.grad.data

        # Project onto L2 ball around original
        delta = x_adv.detach() - x_orig
        norm = torch.norm(delta.flatten())
        if norm > threshold:
            delta = delta * (threshold / norm)
        x_adv = torch.clamp(x_orig + delta, 0.0, 1.0)

        # Early stop if target is predicted
        with torch.no_grad():
            pred = int(torch.argmax(model(x_adv), dim=1).item())
        if pred == target:
            break

    return x_adv.detach().squeeze().numpy()

x_adv = targeted_l2_attack(model, x, target, threshold)
```

### 3. Verify constraint and submit

```python
dist = l2(x_adv, x)
print(f"L2 = {dist:.4f} (threshold: {threshold})")

# Verify local prediction
x_t = torch.from_numpy(x_adv[None, None, ...]).float()
local_pred = int(torch.argmax(model(x_t), dim=1).item())
print(f"Local pred: {local_pred}, target: {target}")

resp = requests.post(f"{BASE_URL}/submit",
    json={"image_b64": b64_png_from_x01(x_adv)}).json()
print(resp)  # {"ok": true, "pred": 6, "target": 6, "l2": 0.68, "flag": "HTB{...}"}
```

## Tuning Tips

- If `l2 > threshold`, reduce `step_size` (0.001–0.003) and increase `num_iter` (300–500). Smaller steps produce tighter perturbations at the cost of more iterations.
- If the prediction doesn't reach the target within `num_iter`, increase step size first, then iterations.
- The server uses dropout in evaluation mode (disabled). Your local model must also be in `.eval()` mode during gradient computation, otherwise predictions are stochastic.
- Server error messages give exact constraint violations: `"L2 too large: 0.82 > 0.75"` or `"Wrong target: predicted 8, need 6"`.

## Lessons Learned

- L2-constrained targeted attacks are harder than L∞ untargeted attacks. The optimizer must steer toward a specific class region, not just escape the current one.
- The DeepFool formulation (orthogonal projection onto the linearized boundary for the target class) is the theoretically optimal approach. The simpler iterative attack above works in practice and is easier to implement under API constraints.
- Projecting onto an L2 ball (rather than an L∞ box) means individual pixels can change by varying amounts. Visually, the perturbation appears as a smooth, low-amplitude haze rather than uniform noise.

## Related Pages

- [[attack/ai/deepfool]]
- [[attack/ai/fgsm]]
- [[attack/ai/i_fgsm]]
- [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]]

## Sources

- raw/ai_evasion_first_order_attacks/deepfool_challenge.md
