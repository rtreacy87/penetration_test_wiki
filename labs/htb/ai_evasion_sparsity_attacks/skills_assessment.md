---
tags: [lab, attack/ai]
module: ai_evasion_sparsity_attacks
last_updated: 2026-05-16
source_count: 1
---

# Skills Assessment — Sparsity Attacks on ResNet-18 CIFAR-10

**Platform:** HTB Academy  
**Module:** AI Evasion — Sparsity Attacks  
**Constraint type:** Mixed (L2 minimum threshold + method signature)  
**Objective:** Craft a targeted adversarial example against a ResNet-18 CIFAR-10 classifier using the method the server specifies (EAD, JSMA, or either).

## Scenario

The server returns one or more challenge items. Each item specifies a CIFAR-10 image, its true class, a target class, and a `required_method` tag (`"ead"`, `"jacobian"`, or `"either"`). You must produce an adversarial image using the specified method and submit it to `/submit_images`. The server validates:

1. The model predicts the target class.
2. The method matches what was required.
3. The L2 distance from the baseline exceeds **1.5** (anti-cheat: prevents submitting the clean image).

The model is ResNet-18 adapted for CIFAR-10 with no initial downsampling stride.

---

## Phase 0 — Environment Setup

```bash
mkdir cifar_sparsity_attack; cd cifar_sparsity_attack
python3 -m venv venv && source venv/bin/activate
pip3 install numpy requests pillow torch torchvision
# matplotlib is listed in the solution but only needed if you add visualization
```

```bash
export BASE_URL="http://<instance_ip>:<port>"
curl -s "$BASE_URL/health" | python3 -m json.tool
# {"status": "ok", "model": "ResNetCIFAR", "items": 1}
```

---

## Phase 1 — API and Challenge Fetch

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/health` | GET | Returns `{"status", "model", "items"}` |
| `/challenge` | GET | Returns `{"items": [{sample_id, label, target, required_method, image_b64}]}` |
| `/model` | GET | Returns `{"arch", "weights_sha256", "weights_url", "normalize"}` |
| `/model/weights` | GET | Download ResNet-18 checkpoint |
| `/predict` | POST | Validate pipeline; returns prediction for a single image |
| `/submit_images` | POST | Submit adversarial images with method tags |

Images are base64-encoded **32×32 RGB PNGs** (`image_b64` field).

```bash
# Download model weights via CLI
curl -sL "$BASE_URL/model/weights" -o cifar10_model.pth
```

```python
import base64, io, json, urllib.request
from dataclasses import dataclass
from typing import List, Dict
import numpy as np
from PIL import Image
import torch
import torch.nn as nn

BASE_URL = "http://<instance_ip>:<port>"
CIFAR10_MEAN = (0.4914, 0.4822, 0.4465)
CIFAR10_STD  = (0.2470, 0.2435, 0.2616)

def cifar_normalize(x: torch.Tensor) -> torch.Tensor:
    """Normalize (N,3,32,32) [0,1] tensor to CIFAR-10 statistics."""
    mean = torch.tensor(CIFAR10_MEAN, dtype=x.dtype, device=x.device)[None, :, None, None]
    std  = torch.tensor(CIFAR10_STD,  dtype=x.dtype, device=x.device)[None, :, None, None]
    return (x - mean) / std

def _to_b64_rgb(x4d: np.ndarray) -> str:
    """Encode (1,3,32,32) [0,1] numpy array to base64 RGB PNG."""
    x    = np.transpose(x4d[0], (1, 2, 0))
    x255 = np.clip((x * 255.0).round(), 0, 255).astype(np.uint8)
    img  = Image.fromarray(x255, mode="RGB")
    buf  = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

def _from_b64_rgb(b64: str) -> np.ndarray:
    """Decode base64 RGB PNG to (1,3,32,32) [0,1] numpy array."""
    raw  = base64.b64decode(b64)
    img  = Image.open(io.BytesIO(raw)).convert("RGB")
    x    = np.asarray(img, dtype=np.float32) / 255.0
    return np.transpose(x, (2, 0, 1))[None, ...].astype(np.float32)

def _http_get(url: str) -> dict:
    with urllib.request.urlopen(url, timeout=30) as r:
        return json.loads(r.read().decode("utf-8"))

def _http_post(url: str, body: dict) -> dict:
    data = json.dumps(body).encode("utf-8")
    req  = urllib.request.Request(url, data=data,
                                  headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=60) as r:
        return json.loads(r.read().decode("utf-8"))

@dataclass
class ChallengeItem:
    sample_id: int
    label: int
    target: int
    required_method: str
    x01: np.ndarray  # (1,3,32,32)

def fetch_challenge(host: str) -> List[ChallengeItem]:
    payload = _http_get(f"{host}/challenge")
    items   = []
    for it in payload["items"]:
        x = _from_b64_rgb(it["image_b64"])
        items.append(ChallengeItem(
            sample_id       = int(it["sample_id"]),
            label           = int(it["label"]),
            target          = int(it["target"]),
            required_method = str(it["required_method"]).lower(),
            x01             = x,
        ))
    return items

items = fetch_challenge(BASE_URL)
for it in items:
    print(f"sample={it.sample_id}  label={it.label}  target={it.target}  method={it.required_method}")
```

---

## Phase 2 — ResNet-18 Architecture and Weight Loading

`ResNetCIFAR` is a ResNet-18 adapted for 32×32 CIFAR-10 inputs. Key difference from standard ResNet-18: the initial conv uses stride=1 and padding=1 (no downsampling), since CIFAR-10 images are already small.

```python
class BasicBlock(nn.Module):
    expansion = 1

    def __init__(self, in_planes: int, planes: int, stride: int = 1) -> None:
        super().__init__()
        self.conv1    = nn.Conv2d(in_planes, planes, 3, stride=stride, padding=1, bias=False)
        self.bn1      = nn.BatchNorm2d(planes)
        self.conv2    = nn.Conv2d(planes, planes, 3, padding=1, bias=False)
        self.bn2      = nn.BatchNorm2d(planes)
        self.shortcut = nn.Sequential()
        if stride != 1 or in_planes != planes:
            self.shortcut = nn.Sequential(
                nn.Conv2d(in_planes, planes, 1, stride=stride, bias=False),
                nn.BatchNorm2d(planes),
            )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        out = torch.relu(self.bn1(self.conv1(x)))
        out = self.bn2(self.conv2(out))
        out += self.shortcut(x)
        return torch.relu(out)


class ResNetCIFAR(nn.Module):
    """ResNet-18 adapted for CIFAR-10 (no initial stride-2 downsample)."""

    def __init__(self, num_blocks=(2, 2, 2, 2), num_classes=10) -> None:
        super().__init__()
        self.in_planes = 64
        self.conv1  = nn.Conv2d(3, 64, 3, stride=1, padding=1, bias=False)
        self.bn1    = nn.BatchNorm2d(64)
        self.layer1 = self._make_layer(64,  num_blocks[0], stride=1)
        self.layer2 = self._make_layer(128, num_blocks[1], stride=2)
        self.layer3 = self._make_layer(256, num_blocks[2], stride=2)
        self.layer4 = self._make_layer(512, num_blocks[3], stride=2)
        self.avgpool = nn.AdaptiveAvgPool2d(1)
        self.fc     = nn.Linear(512, num_classes)

    def _make_layer(self, planes: int, n: int, stride: int) -> nn.Sequential:
        layers = []
        for s in [stride] + [1] * (n - 1):
            layers.append(BasicBlock(self.in_planes, planes, s))
            self.in_planes = planes
        return nn.Sequential(*layers)

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        out = torch.relu(self.bn1(self.conv1(x)))
        out = self.layer1(out);  out = self.layer2(out)
        out = self.layer3(out);  out = self.layer4(out)
        out = self.avgpool(out)
        return self.fc(torch.flatten(out, 1))   # raw logits


def load_model(weights: str, device: torch.device) -> nn.Module:
    ckpt = torch.load(weights, map_location=device)
    # Checkpoint may be a raw state_dict or a dict containing state_dict / state_dict_ema
    if isinstance(ckpt, dict) and ("state_dict" in ckpt or "state_dict_ema" in ckpt):
        sd = ckpt.get("state_dict_ema") or ckpt.get("state_dict")
    else:
        sd = ckpt
    model = ResNetCIFAR().to(device)
    model.load_state_dict(sd)
    return model.eval()

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model  = load_model("cifar10_model.pth", device)
print(f"Model loaded on {device}")
```

---

## Phase 3 — Attack Implementations

### EAD on CIFAR-10

EAD works identically to the MNIST version but uses CIFAR-10 normalization and the C&W margin loss on raw logits. The key difference is the input shape (1,3,32,32 vs 1,1,28,28):

```python
import torch.nn.functional as F

def _ead_adv_loss(logits: torch.Tensor, target: int, confidence: float = 0.0):
    """C&W margin loss: max(score[other] - score[target] + confidence, 0)."""
    target_logit = logits[0, target]
    others = torch.cat([logits[0, :target], logits[0, target+1:]])
    return torch.clamp(others.max() - target_logit + confidence, min=0)

def _shrink_thresh(y, orig, threshold, clip_min=0.0, clip_max=1.0):
    """Soft-thresholding proximal operator."""
    diff     = y - orig
    cond_pos = (diff >  threshold).float()
    cond_zer = (torch.abs(diff) <= threshold).float()
    cond_neg = (diff < -threshold).float()
    return (cond_pos * torch.clamp(y - threshold, min=clip_min, max=clip_max)
          + cond_zer * orig
          + cond_neg * torch.clamp(y + threshold, min=clip_min, max=clip_max))

def ead_targeted_cifar(model, x01: torch.Tensor, target: int,
                       c: float = 0.01, beta: float = 0.01,
                       lr: float = 0.01, max_iter: int = 1000) -> torch.Tensor:
    """
    EAD targeted attack for CIFAR-10.

    Binary search over c with FISTA inner loop.
    Returns best adversarial example found (clipped to [0,1]).
    """
    x_orig = x01.detach().clone().float()
    low, high = 0.0, None
    c_cur = float(c)
    best, best_score = x_orig.clone(), float("inf")

    def _run(c_val):
        adv = x_orig.clone();  y_mom = adv.clone()
        loc_best, loc_score, success = None, float("inf"), False
        for it in range(max_iter):
            y_mom = y_mom.detach().requires_grad_(True)
            logits = model(cifar_normalize(y_mom))
            loss   = c_val * _ead_adv_loss(logits, target) + torch.sum((y_mom - x_orig) ** 2)
            loss.backward()
            y_new = y_mom - lr * y_mom.grad
            adv_n = _shrink_thresh(y_new, x_orig, lr * beta)
            adv_n = torch.clamp(adv_n, 0.0, 1.0)
            mom   = it / (it + 3.0)
            y_mom = torch.clamp(adv_n + mom * (adv_n - adv), 0.0, 1.0)
            adv   = adv_n
            with torch.no_grad():
                pred = int(torch.argmax(model(cifar_normalize(adv)), dim=1).item())
                diff = adv - x_orig
                score = float(torch.norm(diff.flatten()) + beta * torch.sum(torch.abs(diff)))
                if pred == target:
                    success = True
                    if score < loc_score:
                        loc_score, loc_best = score, adv.clone()
                if it > 100 and pred == target and F.softmax(logits, dim=1)[0, target] > 0.9:
                    break
        return (loc_best if loc_best is not None else adv).detach(), success, loc_score

    for _ in range(6):
        cand, ok, score = _run(c_cur)
        if ok:
            if score < best_score:
                best, best_score = cand.clone(), score
            high   = c_cur if high is None else min(high, c_cur)
            c_cur  = (low + (high if high is not None else c_cur)) / 2.0
            if high is not None and (high - low) < 1e-4:
                break
        else:
            low   = c_cur
            c_cur = c_cur * 2.0 if high is None else (low + high) / 2.0
    return torch.clamp(best, 0.0, 1.0).detach()
```

### JSMA on CIFAR-10 (spatial pixel variant)

For RGB images, gradients are computed per channel but the saliency score aggregates all 3 channels per spatial position. This treats each (row, col) location as a single L0 "pixel" even though it has 3 channel values.

Key differences from the MNIST version:
- Input shape: `(1, 3, H, W)` — spatial pixels = H×W = 1024, each with 3 channels
- Gradient aggregation: sum gradients over channels for each spatial position
- Step vector: apply `theta` only to channels where the gradient points in the chosen direction
- Amplitude cap: limit total perturbation per pixel to `AMP_CAP=0.25` across all channels

```python
def jsma_targeted_cifar(model, x01: torch.Tensor, target: int,
                         theta: float = 0.12, gamma: float = 0.15,
                         max_iter: int = 250, top_k: int = 5) -> torch.Tensor:
    """
    JSMA targeted attack for CIFAR-10.

    gamma: fraction of spatial pixels that can be touched (L0 budget = gamma * H * W)
    theta: per-step perturbation magnitude per channel
    AMP_CAP: maximum absolute perturbation per spatial pixel across any binary search run
    """
    torch.manual_seed(1337)
    x      = x01.detach().clone().float()
    B, C, H, W = x.shape
    assert B == 1 and C == 3

    pixel_budget  = max(1, int(gamma * H * W))  # ~15 pixels for 32x32 with gamma=0.15
    touched       = torch.zeros(H * W, dtype=torch.bool, device=x.device)
    saturated     = torch.zeros(H * W, dtype=torch.bool, device=x.device)
    touch_counts  = torch.zeros(H * W, dtype=torch.int32, device=x.device)
    x_orig        = x.clone()
    AMP_CAP, MAX_TOUCHES = 0.25, 5

    changed_unique = 0
    for it in range(max_iter):
        if changed_unique >= pixel_budget:
            break
        x_req = x.detach().clone().requires_grad_(True)
        logits = model(cifar_normalize(x_req))
        pred   = int(torch.argmax(logits, dim=1).item())
        if pred == target:
            break

        # Aggregate gradients over channels to get per-spatial-pixel scores
        g_all = torch.autograd.grad(logits[0].sum(), x_req, retain_graph=True)[0]
        g_t   = torch.autograd.grad(logits[0, target], x_req, retain_graph=False)[0]
        alpha = g_t.view(C, H, W).sum(dim=0).flatten()       # (H*W,)
        beta  = (g_all - g_t).view(C, H, W).sum(dim=0).flatten()

        available = ~saturated
        inc_score = torch.zeros(H * W)
        dec_score = torch.zeros(H * W)
        inc_mask  = (alpha > 0) & (beta < 0) & available
        dec_mask  = (alpha < 0) & (beta > 0) & available
        inc_score[inc_mask] = alpha[inc_mask] * (-beta[inc_mask])
        dec_score[dec_mask] = (-alpha[dec_mask]) * beta[dec_mask]

        # Center-weighting: pixels near mid-range have more room to move
        base_vals = x[0].mean(dim=0).flatten()
        center    = 1.0 - torch.clamp(torch.abs(base_vals - 0.5) / 0.5, 0.0, 1.0)
        inc_score *= center;  dec_score *= center

        # Pick top-k candidates from each direction
        k = min(top_k, pixel_budget - changed_unique)
        scores = []
        if inc_mask.any():
            vals, idxs = torch.topk(inc_score, k=min(k, int(inc_mask.sum())))
            scores += [(float(v), int(i), +1.0) for v, i in zip(vals, idxs) if v > 0]
        if dec_mask.any():
            vals, idxs = torch.topk(dec_score, k=min(k, int(dec_mask.sum())))
            scores += [(float(v), int(i), -1.0) for v, i in zip(vals, idxs) if v > 0]
        scores.sort(key=lambda t: t[0], reverse=True)
        scores = scores[:k]

        if not scores:
            break

        for _, p_idx, direction in scores:
            if saturated[p_idx] or changed_unique >= pixel_budget:
                continue
            r, c_ = p_idx // W, p_idx % W
            # Channel-wise step: only move channels where gradient agrees with direction
            mg = (2 * g_t - g_all)[0, :, r, c_]
            if direction > 0:
                step_vec = (mg > 0).to(x.dtype)
            else:
                step_vec = -(mg < 0).to(x.dtype)
            if float(step_vec.abs().sum()) == 0.0:
                step_vec = torch.full((C,), float(direction))

            candidate = torch.clamp(x[0, :, r, c_] + theta * step_vec, 0.06, 0.94)
            delta      = torch.clamp(candidate - x_orig[0, :, r, c_], -AMP_CAP, AMP_CAP)
            before     = x[0, :, r, c_].clone()
            x[0, :, r, c_] = torch.clamp(x_orig[0, :, r, c_] + delta, 0.06, 0.94)

            if not touched[p_idx] and not torch.allclose(before, x[0, :, r, c_]):
                touched[p_idx] = True;  changed_unique += 1
            touch_counts[p_idx] += 1
            at_cap = float(delta.abs().max()) >= AMP_CAP - 1e-4
            if at_cap or touch_counts[p_idx] >= MAX_TOUCHES:
                saturated[p_idx] = True

    return torch.clamp(x, 0.0, 1.0).detach()
```

---

## Phase 4 — PNG Round-Trip and Submission

Before submitting, perform a PNG encode/decode round-trip to mirror the server's decode path. This ensures your perturbation survives 8-bit quantization:

```python
def png_roundtrip(x4d: np.ndarray) -> torch.Tensor:
    """Quantize through PNG to mirror the server's decode path."""
    b64  = _to_b64_rgb(x4d)
    return torch.from_numpy(_from_b64_rgb(b64))

def craft_adv(model, device, item: ChallengeItem) -> tuple:
    """Craft adversarial example; try JSMA first if method is 'either'."""
    x = torch.from_numpy(item.x01).to(device)
    t = item.target
    m = item.required_method

    if m in ("jacobian", "either"):
        adv = jsma_targeted_cifar(model, x, target=t)
        cq  = png_roundtrip(adv.cpu().numpy())
        with torch.no_grad():
            pred = int(torch.argmax(model(cifar_normalize(cq.to(device))), dim=1).item())
        if pred == t:
            return cq.numpy(), "jacobian"
        if m == "jacobian":
            raise RuntimeError(f"JSMA failed: pred={pred}, target={t}")

    if m in ("ead", "either"):
        for params in [
            {"c": 0.01,  "beta": 0.01, "lr": 0.01, "max_iter": 1000},
            {"c": 0.005, "beta": 0.02, "lr": 0.01, "max_iter": 1200},
            {"c": 0.02,  "beta": 0.01, "lr": 0.005, "max_iter": 800},
        ]:
            adv = ead_targeted_cifar(model, x, target=t, **params)
            cq  = png_roundtrip(adv.cpu().numpy())
            with torch.no_grad():
                pred = int(torch.argmax(model(cifar_normalize(cq.to(device))), dim=1).item())
            if pred == t:
                return cq.numpy(), "ead"
        raise RuntimeError(f"EAD failed for target={t}")

    raise ValueError(f"Unknown method: {m}")

# Run attacks and collect results
advs:    Dict[int, np.ndarray] = {}
methods: Dict[int, str]        = {}
for it in items:
    print(f"[Sample {it.sample_id}] Attacking label={it.label} → target={it.target} method={it.required_method}")
    adv_img, method_used   = craft_adv(model, device, it)
    advs[it.sample_id]    = adv_img
    methods[it.sample_id] = method_used
    print(f"  Done: method={method_used}")

# Submit
submit_items = [
    {"sample_id": it.sample_id, "method": methods[it.sample_id],
     "image_b64": _to_b64_rgb(advs[it.sample_id])}
    for it in items
]
resp = _http_post(f"{BASE_URL}/submit_images", {"items": submit_items})
print(json.dumps(resp, indent=2))
if resp.get("ok"):
    print("Flag:", resp.get("flag"))
```

Expected response:
```json
{
  "ok": true,
  "flag": "HTB{...}",
  "decisions": [
    {
      "sample_id": 0,
      "pred": 8,
      "target": 8,
      "target_ok": true,
      "signature_ok": true,
      "inferred_method": "ead",
      "delta_l2": 2.267,
      "delta_linf": 0.239
    }
  ]
}
```

Validation errors:

| Field | Problem | Fix |
|-------|---------|-----|
| `target_ok: false` | Model doesn't predict target | Increase `max_iter` or try alternate hyperparams |
| `signature_ok: false` | Method tag doesn't match inferred attack type | Submit `"jacobian"` for JSMA, `"ead"` for EAD |
| `delta_l2 < 1.5` | Clean image submitted (or near-zero perturbation) | Ensure you're submitting the adversarial, not the original |

---

## Lessons Learned

- **Method signature detection**: The server infers whether the submitted image looks like EAD or JSMA based on perturbation characteristics (sparsity, L1/L2 ratio, pixel distribution). JSMA produces pixel-sparse noise; EAD produces smooth, small-magnitude noise. Submit the correct `method` tag and let the attack match it.
- **PNG round-trip before submission**: 8-bit quantization can push an adversarial image back across the decision boundary. Always simulate the server's decode path before submitting.
- **CIFAR-10 JSMA sums over channels**: Unlike MNIST (1 channel), each CIFAR-10 spatial position has 3 channels. Sum gradients over channels to get a single saliency score per pixel location; modify channels individually based on per-channel gradient sign.
- **AMP_CAP limits total perturbation per pixel**: The JSMA implementation caps the total delta per spatial pixel at 0.25, regardless of how many times it is visited. This prevents a single pixel from absorbing all the perturbation budget.
- **Checkpoint loading with EMA fallback**: Training pipelines often save both a live `state_dict` and an exponential moving average `state_dict_ema`. The EMA weights typically generalize better. Check for `state_dict_ema` first, then `state_dict`, then fall back to treating the whole checkpoint as a flat state dict.
- **No GUI needed**: The solution imports `matplotlib` but only uses `matplotlib.use("Agg")` to prevent display errors. Remove the import entirely if you don't add visualization code.

---

## Related Pages

- [[attack/ai/jsma]] — JSMA theory: Jacobian saliency, single-pixel vs pairwise, L0 vs L2
- [[attack/ai/elasticnet_attack]] — EAD theory: FISTA, binary search, C&W comparison
- [[labs/htb/ai_evasion_sparsity_attacks/jsma_challenge]] — JSMA on MNIST (LeNet-5)
- [[labs/htb/ai_evasion_sparsity_attacks/elasticnet_challenge]] — EAD on MNIST
- [[labs/htb/ai_evasion_first_order_attacks/skills_assessment_1]] — I-FGSM on CIFAR-10 with same architecture

## Sources

- raw/lab/ai_evasion_sparsity_attacks/skills_assessment.md
