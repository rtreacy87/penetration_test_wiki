---
tags: [lab, attack/ai]
module: ai_evasion_sparsity_attacks
last_updated: 2026-05-16
source_count: 1
---

# ElasticNet Challenge (EAD)

**Platform:** HTB Academy  
**Module:** AI Evasion — Sparsity Attacks  
**Constraint type:** Elastic-net (L2 + β·L1)  
**Objective:** Produce an adversarial MNIST image that causes misclassification while satisfying three simultaneous distance constraints: `elastic_max`, `l2_max`, and `l1_max`.

## Scenario

A server exposes a fixed MNIST classifier and challenges you to craft an adversarial image with minimal combined L1+L2 perturbation. The ElasticNet Attack (EAD) uses FISTA optimization with soft-thresholding to produce perturbations that are simultaneously sparse (small L1 — few pixels significantly changed) and small-magnitude (small L2). A binary search over the regularization constant `C` balances adversarial effectiveness against minimality.

**Three constraints must all hold:**
- `L2 + β·L1 ≤ elastic_max`
- `L2 ≤ l2_max`
- `L1 ≤ l1_max`

---

## Phase 0 — Environment Setup

```bash
mkdir elasticnet_attack; cd elasticnet_attack
python3 -m venv venv && source venv/bin/activate
pip3 install numpy requests pillow torch torchvision
```

Download model weights via CLI:

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
curl -s -o elasticnet_weights.pth "$BASE_URL/weights"
```

---

## Phase 1 — API and Challenge Fetch

| Endpoint     | Method | Purpose                                                                       |
| ------------ | ------ | ----------------------------------------------------------------------------- |
| `/health`    | GET    | Service check (`{"status": "ok"}`)                                            |
| `/challenge` | GET    | Returns `{label, beta, elastic_max, l2_max, l1_max, sample_index, image_b64}` |
| `/weights`   | GET    | Download model `state_dict` (binary `.pth`)                                   |
| `/submit`    | POST   | Validate all three distance constraints + misclassification; returns flag     |

```python
import base64, io, json, time
from dataclasses import dataclass
import numpy as np
import requests
from PIL import Image
import torch
import torch.nn as nn

BASE_URL   = "http://<instance_ip>:<port>"
MNIST_MEAN = 0.1307
MNIST_STD  = 0.3081

def x01_from_b64_png(b64: str) -> np.ndarray:
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0.0, 1.0)

def b64_png_from_x01(x2d: np.ndarray) -> str:
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img   = Image.fromarray(x255, mode="L")
    buf   = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

def mnist_normalize(x01: torch.Tensor) -> torch.Tensor:
    return (x01 - MNIST_MEAN) / MNIST_STD

@dataclass
class Challenge:
    label: int
    beta: float
    elastic_max: float
    l2_max: float
    l1_max: float
    sample_index: int
    x01: np.ndarray  # (1,1,28,28)

def fetch_challenge(host: str) -> Challenge:
    r = requests.get(f"{host}/challenge", timeout=10)
    r.raise_for_status()
    p    = r.json()
    x2d  = x01_from_b64_png(p["image_b64"])
    x4d  = x2d[None, None, ...].astype(np.float32)
    return Challenge(
        label        = int(p["label"]),
        beta         = float(p["beta"]),
        elastic_max  = float(p["elastic_max"]),
        l2_max       = float(p["l2_max"]),
        l1_max       = float(p["l1_max"]),
        sample_index = int(p["sample_index"]),
        x01          = x4d,
    )

chall = fetch_challenge(BASE_URL)
print(f"label={chall.label}  beta={chall.beta}")
print(f"elastic_max={chall.elastic_max}  l2_max={chall.l2_max}  l1_max={chall.l1_max}")
```

---

## Phase 2 — Model Architecture

EAD uses the same `SimpleClassifier` as the FGSM/DeepFool challenges, but **the forward method returns raw logits** (no `log_softmax`). The C&W-style loss in EAD operates directly on logits via a margin formulation.

```python
class SimpleClassifier(nn.Module):
    def __init__(self) -> None:
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
        return self.fc2(x)    # raw logits — NOT log_softmax

def load_model(weights_path: str) -> nn.Module:
    model = SimpleClassifier()
    model.load_state_dict(torch.load(weights_path, map_location="cpu"))
    return model.eval()

model = load_model("elasticnet_weights.pth")
```

---

## Phase 3 — EAD with FISTA Optimization

### Algorithm overview

EAD combines two ideas from optimization:

1. **C&W-style adversarial loss** — maximize the margin between the true class and the best competing class:
   ```
   L_adv = max(logit[true] − max_{k≠true} logit[k], 0)
   ```
   This is zero when already misclassified; positive otherwise.

2. **Elastic-net regularization** — minimize `L2 + β·L1` to keep the perturbation small and sparse:
   ```
   L_reg = L2 + β·L1
   ```
   β controls the sparsity/magnitude trade-off: high β → sparser (more L1 push); low β → smoother (more L2 push).

3. **FISTA** (Fast Iterative Shrinkage-Thresholding Algorithm) — gradient descent + soft-thresholding with Nesterov momentum:
   - Gradient step: `y_new = y - lr · ∇(L_adv + L2)`
   - Proximal step (soft thresholding): snap values within β of the original back to original; shift others by β
   - Momentum: `y_mom = x_new + (step/(step+3)) · (x_new − x_prev)`

4. **Binary search over C** — C scales how much the adversarial loss dominates. Too low: regularization wins, attack may fail. Too high: attack succeeds but perturbation is unnecessarily large. Binary search converges to the smallest C that still causes misclassification.

### Loss function

```python
def _loss(model, x: torch.Tensor, orig: torch.Tensor,
          y_onehot: torch.Tensor, const: torch.Tensor, beta: float):
    """
    C&W adversarial term + elastic-net regularization.

    The C&W term uses a one-hot mask to extract the true class logit and the
    maximum competing logit. Penalizes by the margin.
    """
    logits    = model(mnist_normalize(x))
    l1        = torch.sum(torch.abs(x - orig), dim=(1, 2, 3))
    l2        = torch.sum((x - orig) ** 2, dim=(1, 2, 3))
    elastic   = l2 + beta * l1
    real      = torch.sum(y_onehot * logits, dim=1)          # true class score
    other     = torch.max((1 - y_onehot) * logits - y_onehot * 1e4, dim=1)[0]  # best competitor
    adv_loss  = torch.clamp(real - other, min=0)             # zero when misclassified
    total     = torch.sum(const * adv_loss) + torch.sum(l2) + beta * torch.sum(l1)
    return total, l1, l2, elastic
```

### Proximal operator (soft thresholding)

The soft-thresholding operator enforces sparsity: perturbations smaller than `β` are collapsed back to the original value (true zeros in the perturbation), larger ones are shrunk by `β`.

```python
def _prox(x: torch.Tensor, y: torch.Tensor, orig: torch.Tensor,
          beta: float, step: int):
    """
    Proximal update + FISTA momentum.

    Values within beta of original → snap back to original (zero perturbation).
    Values beyond beta of original → shift toward original by beta.
    Momentum parameter zt = step / (step + 3) follows Nesterov acceleration schedule.
    """
    zt   = step / (step + 3.0)
    diff = y - orig
    cond_pos = (diff >  beta).float()
    cond_zer = (torch.abs(diff) <= beta).float()
    cond_neg = (diff < -beta).float()
    upper   = torch.minimum(y - beta, torch.tensor(1.0))
    lower   = torch.maximum(y + beta, torch.tensor(0.0))
    x_new   = cond_pos * upper + cond_zer * orig + cond_neg * lower
    y_new   = x_new + zt * (x_new - x)        # Nesterov momentum
    return x_new, y_new
```

### Full EAD attack with binary search

```python
def ead_attack(model, x01: np.ndarray, label: int, beta: float,
               max_iter: int = 400, lr: float = 1e-2, bin_steps: int = 5,
               initial_const: float = 1e-3) -> np.ndarray:
    """
    EAD: binary search over C with FISTA inner loop.

    bin_steps: number of binary search iterations over C
    max_iter: FISTA iterations per C value
    initial_const: starting value of C (scales adversarial loss weight)
    """
    device = torch.device("cpu")
    orig   = torch.from_numpy(x01).to(device)
    y      = torch.tensor([label], device=device, dtype=torch.long)
    y_oh   = torch.zeros(1, 10, device=device)
    y_oh.scatter_(1, y.view(-1, 1), 1)

    lower = torch.zeros(1, device=device)
    upper = torch.ones(1, device=device) * 1e10
    const = torch.ones(1, device=device) * initial_const
    best  = orig.clone()
    best_dist = torch.tensor(1e10)

    for b in range(bin_steps):
        x  = orig.clone()
        yv = orig.clone()           # y variable in FISTA (momentum iterate)
        for it in range(max_iter):
            yv.requires_grad_(True)
            total, l1, l2, elastic = _loss(model, yv, orig, y_oh, const, beta)
            grad = torch.autograd.grad(total, yv)[0]
            with torch.no_grad():
                yv -= lr * grad
                x, yv = _prox(x, yv, orig, beta, it + 1)
            x.clamp_(0.0, 1.0);  yv.clamp_(0.0, 1.0)

            # Check for success every 100 steps
            if it % 100 == 0:
                with torch.no_grad():
                    pred = int(torch.argmax(model(mnist_normalize(x)), dim=1).item())
                    if pred != label and elastic[0] < best_dist:
                        best_dist = elastic[0].clone()
                        best = x.clone()

        # Binary search update
        with torch.no_grad():
            pred = int(torch.argmax(model(mnist_normalize(x)), dim=1).item())
            if pred != label:
                upper[0] = min(upper[0], const[0])
                const[0] = (lower[0] + upper[0]) / 2 if upper[0] < 1e9 else const[0]
            else:
                lower[0] = max(lower[0], const[0])
                const[0] = ((lower[0] + upper[0]) / 2 if upper[0] < 1e9
                             else const[0] * 10)

    return best.detach().cpu().numpy()
```

---

## Phase 4 — Validate and Submit

```python
torch.manual_seed(1337)
x_adv = ead_attack(model, chall.x01, chall.label, chall.beta)

# Compute all three distances
diff    = x_adv - chall.x01
l1      = float(np.sum(np.abs(diff)))
l2      = float(np.sqrt(np.sum(diff ** 2)))
elastic = l2 + chall.beta * l1
adv_pred = int(torch.argmax(
    model(mnist_normalize(torch.from_numpy(x_adv))), dim=1).item())

print(f"clean={chall.label}  adv_pred={adv_pred}")
print(f"L1={l1:.3f} (max: {chall.l1_max})  L2={l2:.3f} (max: {chall.l2_max})")
print(f"elastic={elastic:.3f} (max: {chall.elastic_max})")

# Submit
b64  = b64_png_from_x01(x_adv[0, 0])
resp = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64}, timeout=30)
resp.raise_for_status()
print("Flag:", resp.json().get("flag"))
```

Expected output:
```
clean=9  adv_pred=8
L1=10.739  L2=1.298  elastic=1.405 (max: ~1.5)
Flag: HTB{...}
```

Validation errors:

| HTTP status | Failure mode | Fix |
|-------------|-------------|-----|
| 400 | `"elastic_max violated"` | Increase `bin_steps` (more binary search) or decrease `beta` |
| 400 | `"l2_max violated"` | Reduce `lr` or increase `max_iter` |
| 400 | `"l1_max violated"` | Increase `beta` to push harder for sparsity |
| 400 | `"Misclassification not achieved"` | Increase `initial_const` (start with stronger adversarial push) |

---

## Lessons Learned

- **Raw logits vs log-softmax**: `SimpleClassifier` here returns raw logits, not log-probabilities. The C&W margin loss operates directly on logits (`real − other`), so `nll_loss` (which requires log-probabilities) would be wrong here.
- **FISTA vs plain gradient descent**: Standard gradient descent on the elastic-net objective would be slow because L1 is not smooth. FISTA handles this by separating the smooth part (gradient step on L2 + adversarial loss) from the non-smooth part (proximal/thresholding step for L1). The result converges much faster.
- **Proximal operator is the key to sparsity**: The soft-thresholding step snaps perturbations smaller than β back to zero. Without it, L1 minimization degenerates to a smooth approximation and you get noise spread across all pixels rather than a sparse attack.
- **Binary search over C**: C controls how hard the attack "tries" to misclassify versus staying close to the original. Too low C means the regularization term wins and the model is never fooled. The binary search finds the minimum C that achieves misclassification, producing the smallest perturbation that still works.
- **Set a fixed seed**: EAD's performance can vary between runs due to numerical sensitivity. `torch.manual_seed(1337)` before `ead_attack()` makes the result reproducible.

---

## Related Pages

- [[attack/ai/elasticnet_attack]] — full EAD theory: FISTA derivation, EN vs L1 decision rules, C&W comparison
- [[attack/ai/adversarial_examples]] — L1/L2/L∞/L0 norm taxonomy
- [[labs/htb/ai_evasion_sparsity_attacks/jsma_challenge]] — L0-constrained attack on same platform
- [[labs/htb/ai_evasion_sparsity_attacks/skills_assessment]] — EAD on ResNet-18 CIFAR-10

## Sources

- raw/lab/ai_evasion_sparsity_attacks/elasticnet_challenge.md
