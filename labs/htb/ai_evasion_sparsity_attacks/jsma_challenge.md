---
tags: [lab, attack/ai]
module: ai_evasion_sparsity_attacks
last_updated: 2026-05-16
source_count: 1
---

# JSMA Challenge

**Platform:** HTB Academy  
**Module:** AI Evasion — Sparsity Attacks  
**Constraint type:** L0  
**Objective:** Craft a targeted adversarial MNIST image by modifying at most `l0_budget` pixels; the model must predict the target class.

## Scenario

A server exposes a fixed LeNet-5 MNIST classifier. You receive a baseline image, its true label, a target class, and an L0 pixel budget. Implement JSMA with pairwise saliency to select and modify the fewest pixels needed to steer the classifier toward the target class, then submit via API.

**Constraints:**
- `‖x_adv − x‖₀ ≤ l0_budget` — at most `l0_budget` pixels changed
- All pixel values remain in `[0, 1]`

---

## Phase 0 — Environment Setup

```bash
mkdir jsma_attack; cd jsma_attack
python3 -m venv venv && source venv/bin/activate
pip3 install numpy requests pillow torch torchvision
```

---

## Phase 1 — API and Challenge Fetch

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
# {"status": "healthy"}
curl -s "$BASE_URL/challenge" | python3 -m json.tool
```

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Service check |
| `/challenge` | GET | Returns `{target_class, l0_budget, original_label, sample_index, image_b64}` |
| `/weights` | GET | Download model `state_dict` |
| `/submit` | POST | Validate L0 constraint + target class; returns flag |

```python
import base64, io, json, time
import numpy as np
import requests
from PIL import Image
from dataclasses import dataclass
from typing import Optional

BASE_URL   = "http://<instance_ip>:<port>"
MNIST_MEAN = 0.1307
MNIST_STD  = 0.3081

def x01_from_b64_png(b64: str) -> np.ndarray:
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.clip(np.asarray(img, dtype=np.float32) / 255.0, 0.0, 1.0)

def png_from_x01(x2d: np.ndarray) -> str:
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img   = Image.fromarray(x255, mode="L")
    buf   = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

@dataclass
class Challenge:
    target_class: int
    l0_budget: int
    original_label: int
    sample_index: int
    x01: np.ndarray  # (1,1,28,28)

def fetch_challenge(host: str) -> Challenge:
    r = requests.get(f"{host}/challenge", timeout=10)
    r.raise_for_status()
    p    = r.json()
    x2d  = x01_from_b64_png(p["image_b64"])
    x4d  = x2d[None, None, ...].astype(np.float32)
    return Challenge(
        target_class   = int(p["target_class"]),
        l0_budget      = int(p["l0_budget"]),
        original_label = int(p["original_label"]),
        sample_index   = int(p["sample_index"]),
        x01            = x4d,
    )

chall = fetch_challenge(BASE_URL)
print(f"label={chall.original_label}  target={chall.target_class}  budget={chall.l0_budget}")
```

---

## Phase 2 — Model Architecture and Weight Loading

This challenge uses a LeNet-5 style network with **Tanh activations** and **average pooling** — not ReLU or max-pool. Using the wrong architecture will produce incorrect gradients and a broken attack.

```python
import torch
import torch.nn as nn
import torch.nn.functional as F

class MNISTClassifier(nn.Module):
    """LeNet-5: Conv→AvgPool→Conv→AvgPool→FC×3 with Tanh activations."""

    def __init__(self) -> None:
        super().__init__()
        self.conv1 = nn.Conv2d(1, 6,  kernel_size=5)   # (B,1,28,28) → (B,6,24,24)
        self.conv2 = nn.Conv2d(6, 16, kernel_size=5)   # (B,6,12,12) → (B,16,8,8)
        self.pool  = nn.AvgPool2d(kernel_size=2, stride=2)
        self.fc1   = nn.Linear(16 * 4 * 4, 120)
        self.fc2   = nn.Linear(120, 84)
        self.fc3   = nn.Linear(84, 10)
        self.act   = nn.Tanh()

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        x = self.act(self.conv1(x));  x = self.pool(x)   # (B,6,12,12)
        x = self.act(self.conv2(x));  x = self.pool(x)   # (B,16,4,4)
        x = torch.flatten(x, 1)                          # (B,256)
        x = self.act(self.fc1(x));    x = self.act(self.fc2(x))
        x = self.fc3(x)
        return F.log_softmax(x, dim=1)

def mnist_normalize(x01: torch.Tensor) -> torch.Tensor:
    return (x01 - MNIST_MEAN) / MNIST_STD

# Download weights
wt_bytes = requests.get(f"{BASE_URL}/weights", timeout=30).content
with open("jsma_weights.pth", "wb") as f:
    f.write(wt_bytes)

model = MNISTClassifier()
model.load_state_dict(torch.load("jsma_weights.pth", map_location="cpu"))
model.eval()

# Sanity check
x_t = torch.from_numpy(chall.x01)
pred = int(torch.argmax(model(mnist_normalize(x_t)), dim=1).item())
print(f"Clean pred: {pred}  (true label: {chall.original_label})")
```

---

## Phase 3 — JSMA with Pairwise Saliency

### Single-pixel saliency

At each iteration, compute the full Jacobian (∂F_k/∂x_i for all classes k and pixels i) and score each pixel by how much it helps move toward the target while hurting other classes:

**Saliency score (increase direction):**
```
S_inc(i) = α_i · |β_i|    when  α_i > 0  and  β_i < 0
         = 0               otherwise
```
Where `α_i = ∂F_target/∂x_i` and `β_i = Σ_{k≠target} ∂F_k/∂x_i`.

- Increase: add `theta` to pixel i (valid when increasing x_i helps target and hurts others)
- Decrease: subtract `theta` from pixel i (valid when `α_i < 0` and `β_i > 0`)

### Pairwise saliency

When no single pixel has valid saliency (all α_i·β_i conditions fail), pairs of pixels can be jointly valid: if pixel p raises the target gradient but also raises others, and pixel q is the opposite, their combined saliency may satisfy the direction criteria. This prevents the attack from stalling early:

```python
# Pair (p, q) is valid for increase when:
#   (α_p + α_q) > 0  AND  (β_p + β_q) < 0
# Score = (α_p + α_q) * |(β_p + β_q)|
```

```python
def compute_jacobian(model, x: torch.Tensor) -> torch.Tensor:
    """Return Jacobian (10, 784): row k is ∂F_k/∂x for each of 10 classes."""
    x_flat   = x.view(-1)
    jacobian = torch.zeros(10, x_flat.shape[0])
    for k in range(10):
        if x.grad is not None:
            x.grad.zero_()
        logits = model(mnist_normalize(x))
        logits[0, k].backward(retain_graph=True)
        jacobian[k] = x.grad.view(-1).clone()
    return jacobian


def compute_saliency_map(jacobian, target, search_space):
    """Single-pixel saliency scores for increase and decrease directions."""
    target_grad    = jacobian[target]
    other_grad_sum = jacobian.sum(dim=0) - target_grad

    n = jacobian.shape[1]
    saliency_inc = torch.zeros(n)
    saliency_dec = torch.zeros(n)

    mask_inc = (target_grad > 0) & (other_grad_sum < 0) & search_space
    mask_dec = (target_grad < 0) & (other_grad_sum > 0) & search_space

    saliency_inc[mask_inc] = target_grad[mask_inc] * torch.abs(other_grad_sum[mask_inc])
    saliency_dec[mask_dec] = torch.abs(target_grad[mask_dec]) * other_grad_sum[mask_dec]
    return saliency_inc, saliency_dec


def compute_pairwise_saliency(jacobian, target, search_space,
                               direction: str, top_k: int = 128):
    """
    Find the pixel pair with the highest combined saliency score.

    To avoid O(784²) exhaustive search, pre-filter to the top_k candidates
    by individual |α|·|β| magnitude before evaluating all pairs.

    Returns (p_idx, q_idx, score) or (-1, -1, 0.0) if no valid pair.
    """
    valid = torch.nonzero(search_space, as_tuple=False).squeeze(1)
    if valid.numel() < 2:
        return -1, -1, 0.0

    target_grad = jacobian[target]
    other_grad  = jacobian.sum(dim=0) - target_grad

    if top_k and valid.numel() > top_k:
        prelim  = torch.abs(target_grad[valid]) * torch.abs(other_grad[valid])
        _, tidx = torch.topk(prelim, top_k)
        valid   = valid[tidx]

    best_score = 0.0
    best_pair  = (-1, -1)
    for i in range(valid.numel()):
        p = int(valid[i].item())
        for j in range(i + 1, valid.numel()):
            q         = int(valid[j].item())
            alpha_sum = target_grad[p] + target_grad[q]
            beta_sum  = other_grad[p]  + other_grad[q]
            if direction == "increase":
                if alpha_sum <= 0 or beta_sum >= 0:
                    continue
                score = float(alpha_sum * torch.abs(beta_sum))
            else:
                if alpha_sum >= 0 or beta_sum <= 0:
                    continue
                score = float(torch.abs(alpha_sum) * beta_sum)
            if score > best_score:
                best_score, best_pair = score, (p, q)
    if best_pair[0] == -1:
        return -1, -1, 0.0
    return best_pair[0], best_pair[1], best_score
```

### Main attack loop

```python
def jsma_targeted(model, x01: np.ndarray, target_class: int,
                   l0_budget: int, theta: float = 1.0,
                   max_iters: int = 2000) -> np.ndarray:
    """
    Theta=1.0 immediately saturates pixels (sets them to max brightness).
    This is the most efficient strategy: fewer steps but each step uses the
    full budget per pixel. Reduce theta if you want gradual changes.
    """
    x_orig = torch.from_numpy(x01.copy()).float()
    x_adv  = x_orig.clone()
    search_space = torch.ones(28 * 28, dtype=torch.bool)  # all pixels valid initially

    for iteration in range(max_iters):
        with torch.no_grad():
            pred = int(torch.argmax(model(mnist_normalize(x_adv)), dim=1).item())
        if pred == target_class:
            print(f"Success at iteration {iteration}: pred={target_class}")
            break

        x_diff          = torch.abs(x_adv - x_orig).view(-1)
        pixels_modified = int((x_diff > 1e-6).sum().item())
        if pixels_modified >= l0_budget:
            print(f"L0 budget exhausted: {pixels_modified} pixels modified")
            break

        # Compute Jacobian and saliency
        x_grad   = x_adv.clone().requires_grad_(True)
        jacobian = compute_jacobian(model, x_grad)
        sal_inc, sal_dec = compute_saliency_map(jacobian, target_class, search_space)

        # Try pairwise first; fall back to single-pixel
        pi_p, pi_q, pi_s = compute_pairwise_saliency(jacobian, target_class, search_space, "increase")
        pd_p, pd_q, pd_s = compute_pairwise_saliency(jacobian, target_class, search_space, "decrease")

        if pi_s > pd_s and pi_p >= 0:
            pair, step = (pi_p, pi_q), +theta
        elif pd_s > 0 and pd_p >= 0:
            pair, step = (pd_p, pd_q), -theta
        else:
            pair, step = None, None

        budget_remaining = l0_budget - pixels_modified
        current_mask     = x_diff > 1e-6
        if pair is not None:
            required = sum(1 for idx in pair if not bool(current_mask[idx].item()))
            if required > budget_remaining:
                pair = None

        x_flat = x_adv.view(-1)
        if pair is not None:
            for idx in pair:
                x_flat[idx] = torch.clamp(x_flat[idx] + step, 0.0, 1.0)
                # Remove from search space once saturated
                if x_flat[idx] <= 0.0 or x_flat[idx] >= 1.0:
                    search_space[idx] = False
        else:
            # Fall back to best single pixel
            if sal_inc.max() == 0 and sal_dec.max() == 0:
                print(f"No valid pixels at iteration {iteration}")
                break
            if sal_inc.max() >= sal_dec.max():
                idx = int(sal_inc.argmax().item());   x_flat[idx] = torch.clamp(x_flat[idx] + theta, 0.0, 1.0)
            else:
                idx = int(sal_dec.argmax().item());   x_flat[idx] = torch.clamp(x_flat[idx] - theta, 0.0, 1.0)
            if x_flat[idx] <= 0.0 or x_flat[idx] >= 1.0:
                search_space[idx] = False
        x_adv = x_flat.view(1, 1, 28, 28)

    return x_adv.detach().cpu().numpy()
```

---

## Phase 4 — Validate and Submit

```python
x_adv = jsma_targeted(model, chall.x01, chall.target_class, chall.l0_budget)

# Verify locally
adv_pred = int(torch.argmax(
    model(mnist_normalize(torch.from_numpy(x_adv))), dim=1).item())
l0_used  = int(np.sum(np.abs(x_adv - chall.x01) > 1e-6))
print(f"Adversarial pred: {adv_pred}  (target: {chall.target_class})")
print(f"L0 used: {l0_used} / {chall.l0_budget}")

# Submit
b64  = png_from_x01(x_adv[0, 0])
resp = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64}, timeout=10)
resp.raise_for_status()
print(resp.json())
# {"ok": true, "flag": "HTB{...}"}
```

Expected output:
```
Original prediction: 1, True label: 1
Target class: 7, L0 budget: 50
Success at iteration 22: pred=7
Adversarial pred: 7  L0 used: 18 / 50
```

Validation errors:

| HTTP status | Message | Fix |
|-------------|---------|-----|
| 400 | `"L0 constraint violated"` | More pixels changed than budget; count with `|diff| > 1e-6` |
| 400 | `"Target class not achieved"` | Model still predicts wrong class; increase `max_iters` or reduce `theta` |
| 400 | `"No valid saliency"` | All pixels saturated or search space empty; switch to pairwise only |

---

## Lessons Learned

- **LeNet-5 uses Tanh + AvgPool, not ReLU + MaxPool**: Tanh has gradients everywhere (no dead neurons), which improves saliency quality. Using the wrong activation results in incorrect Jacobians and a broken attack.
- **Pairwise saliency extends the attack's reach**: A pixel where `α_i > 0` but `β_i > 0` (invalid alone) may form a valid pair with another pixel where the combined `β` sum turns negative. Without pairwise JSMA, the attack stalls much earlier.
- **Saturated pixels must be pruned**: Once a pixel reaches 0.0 or 1.0, further perturbation is impossible. Removing it from the search space prevents wasted gradient computations.
- **theta=1.0 is the most efficient theta**: It saturates a pixel in one step (max brightness), maximising the gradient effect while counting as a single L0 step. Smaller theta requires multiple steps per pixel, burning more of the budget.
- **10 backward passes per iteration**: The Jacobian loop runs one backward pass per class. For MNIST (10 classes, 28×28=784 pixels), each iteration is relatively cheap. On larger models, use `torch.autograd.grad` with output indexing to reduce overhead.

---

## Related Pages

- [[attack/ai/jsma]] — full JSMA theory: single-pixel saliency math, pairwise extension, L0 vs L2 trade-off
- [[attack/ai/adversarial_examples]] — L0/L2/L∞ norm taxonomy
- [[labs/htb/ai_evasion_jsma_challenge]] — earlier JSMA lab (simplified single-pixel implementation)
- [[labs/htb/ai_evasion_sparsity_attacks/elasticnet_challenge]] — EAD on the same MNIST platform
- [[labs/htb/ai_evasion_sparsity_attacks/skills_assessment]] — JSMA on ResNet-18 CIFAR-10

## Sources

- raw/lab/ai_evasion_sparsity_attacks/jsma_challenge.md
