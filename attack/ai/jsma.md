---
tags: [attack, attack/ai]
module: ai_evasion_sparsity
last_updated: 2026-05-10
source_count: 12
---

# Jacobian-Based Saliency Map Attack (JSMA)

A white-box, L0-constrained adversarial attack that uses the model's input Jacobian to identify and modify the fewest possible pixels needed to force targeted misclassification.

## Overview

JSMA was introduced by Papernot et al. (2016) in "The Limitations of Deep Learning in Adversarial Settings." It minimizes the L0 norm — the count of modified features — rather than perturbation magnitude. Individual pixels can jump dramatically (black to white) if the saliency map identifies them as strategically valuable. On MNIST, effective configurations modify only 20-40 pixels out of 784 while achieving targeted misclassification. This creates visually apparent dots or strokes rather than imperceptible noise, trading stealth for extreme sparsity.

## Key Concepts and Techniques

### The Jacobian Matrix

The foundation of JSMA is the input Jacobian: a matrix where entry `[i, j]` records how much output class `i` changes when input pixel `j` increases by a small amount. For a model with `m` classes and `n` input pixels, the Jacobian has shape `(m, n)`. Computing it requires one backward pass per output class — 10 backward passes for MNIST.

Gradients must be computed with respect to **logits** (pre-softmax scores), not probabilities. After softmax, probabilities must sum to 1, which mathematically forces the gradient of all competitors to equal the negative of the target gradient. This collapses the saliency formula's competitor suppression term entirely, breaking the attack.

### Saliency Scoring

Given the Jacobian, JSMA extracts two signals for each pixel:
- **Alpha (target gradient):** how much increasing that pixel raises the target class score.
- **Beta (competitor gradient):** the summed effect on all other classes.

Beta is computed efficiently as `sum(all gradients) - target gradient`, avoiding a loop over classes.

The saliency score for modifying a pixel in the **increase** direction is:
```
score = alpha * |beta|  when alpha > 0 AND beta < 0
score = 0               otherwise
```

The sign constraint encodes the goal: a good pixel boosts the target (`alpha > 0`) while suppressing competitors (`beta < 0`). For the **decrease** direction, signs flip: `alpha < 0` AND `beta > 0`, meaning decreasing that pixel raises the target while reducing competitors. Both directions are scored and the higher-scoring pixel wins.

Intuition: if modifying pixel `j` would boost class 2's score by 0.6 while dropping all other classes by 0.4 combined, the saliency score is `0.6 * 0.4 = 0.24`. A pixel that helps the target but also helps competitors gets score 0 — JSMA rejects it.

### Search Space Management

The search space is a boolean mask tracking which pixels remain modifiable. A pixel is removed from the search space when:
1. It is immediately masked after selection (prevents reselection in single-pixel mode).
2. It saturates at `clip_min` (0.0) or `clip_max` (1.0) — further modification in that direction is impossible.

Saturation detection uses epsilon tolerance (`1e-6`) to handle floating-point rounding.

### Iterative Structure

Unlike single-step attacks (FGSM), JSMA iterates a complete decision cycle at each step:
1. Compute the Jacobian matrix (m backward passes).
2. Extract alpha and beta gradients; apply search space mask.
3. Score all pixels in both directions; select winner.
4. Apply perturbation with step size theta; clamp to [0, 1].
5. Update search space (remove selected pixel; scan for saturated pixels).
6. Check if target class now wins; stop if yes or if pixel budget exhausted.

Two parameters control behavior:
- **theta:** perturbation magnitude per modification. `theta=1.0` saturates pixels immediately (aggressive, fast). `theta=0.25` creates subtler changes but may need many more iterations.
- **gamma:** pixel budget as a fraction of total pixels. `gamma=0.15` on MNIST allows `0.15 * 784 = 117` pixel modifications.

## Single-Pixel JSMA

The simplest variant selects one pixel per iteration. This is the baseline implementation that demonstrates saliency mechanics most clearly.

### Key Functions

```python
def compute_class_gradient(x, model, class_idx, wrt='logits'):
    x_grad = x.detach().requires_grad_(True)
    logits = model(x_grad)
    if wrt == 'logits':
        scalar = logits[0, class_idx]
    else:
        probs = F.softmax(logits, dim=1)
        scalar = probs[0, class_idx]
    scalar.backward()
    grad = x_grad.grad.detach().cpu().numpy().flatten().copy()
    return grad

def compute_jacobian_matrix(x, model, num_classes=10, wrt='logits'):
    # Requires batch size 1 — batched inputs average gradients incorrectly
    if x.shape[0] != 1:
        raise ValueError("compute_jacobian_matrix expects batch size 1")
    jacobian = []
    for class_idx in range(num_classes):
        grad = compute_class_gradient(x, model, class_idx, wrt)
        jacobian.append(grad)
    return np.asarray(jacobian)  # shape: (num_classes, num_pixels)
```

```python
def score_increase_saliency(target_grad, other_grad):
    increase_mask = (target_grad > 0) & (other_grad < 0)
    scores = target_grad * np.abs(other_grad) * increase_mask
    return scores

def score_decrease_saliency(target_grad, other_grad):
    decrease_mask = (target_grad < 0) & (other_grad > 0)
    scores = np.abs(target_grad) * other_grad * decrease_mask
    return scores

def select_best_direction(inc_scores, dec_scores):
    max_inc_idx = int(np.argmax(inc_scores))
    max_dec_idx = int(np.argmax(dec_scores))
    max_inc_score = float(inc_scores[max_inc_idx])
    max_dec_score = float(dec_scores[max_dec_idx])
    if max_inc_score > max_dec_score:
        return max_inc_idx, max_inc_score, True   # True = increase direction
    else:
        return max_dec_idx, max_dec_score, False  # False = decrease direction
```

```python
def apply_single_pixel_perturbation(x, pixel_idx, theta, increase, clip_min=0.0, clip_max=1.0):
    original_shape = x.shape
    x_flat = x.view(-1).clone()
    perturbation = theta if increase else -theta
    x_flat[pixel_idx] = torch.clamp(x_flat[pixel_idx] + perturbation, clip_min, clip_max)
    x_modified = x_flat.view(original_shape)
    return x_modified
```

### Single-Pixel Attack Loop

```python
# Configuration
config = {'theta': 0.25, 'gamma': 0.15, 'max_iter': 100, 'wrt': 'logits',
          'clip_min': 0.0, 'clip_max': 1.0}
num_features = int(np.prod(x.shape[1:]))
max_pixels = int(config['gamma'] * num_features)  # e.g., 117 for MNIST

x_adv = x.clone().detach()
search_space = np.ones(num_features, dtype=bool)
pixels_modified = 0

for iteration in range(config['max_iter']):
    if check_target_reached(x_adv, target_class, model):
        break
    if pixels_modified >= max_pixels:
        break

    jacobian = compute_jacobian_matrix(x_adv, model, 10, config['wrt'])
    alpha = extract_target_gradient(jacobian, target_class)
    beta  = extract_other_gradients(jacobian, target_class)
    alpha_masked = apply_search_mask(alpha, search_space)
    beta_masked  = apply_search_mask(beta, search_space)

    inc_scores = score_increase_saliency(alpha_masked, beta_masked)
    dec_scores = score_decrease_saliency(alpha_masked, beta_masked)
    pixel_idx, saliency, increase = select_best_direction(inc_scores, dec_scores)

    if saliency <= 0:
        break  # no valid pixels remain

    x_adv = apply_single_pixel_perturbation(
        x_adv, pixel_idx, config['theta'], increase,
        config['clip_min'], config['clip_max'])

    search_space[pixel_idx] = False
    search_space = remove_saturated_pixels(search_space, x_adv, 0.0, 1.0)
    pixels_modified += 1
```

### Single-Pixel Performance

On MNIST with `theta=0.25`, `gamma=0.15`, 10 samples: 70% success rate, average 73.6 pixels modified (9.39% of image). Step size experiments reveal a threshold effect: `theta=1.0` succeeds where 0.25, 0.50, and 0.10 all fail on the same sample. With `theta=1.0`, each pixel saturates immediately (0.0 → 1.0), enabling misclassification with fewer total modifications.

## Pairwise JSMA

The canonical algorithm from the original paper modifies two pixels simultaneously. Pairwise saliency captures feature interactions that single-pixel selection misses by scoring pixel pairs rather than individuals.

### Pairwise Saliency

For a candidate pair (p, q), the combined gradients are:
```
alpha_pq = alpha_p + alpha_q
beta_pq  = beta_p  + beta_q
```

The pair score for increasing both features is:
```
score = alpha_pq * |beta_pq|  when alpha_pq > 0 AND beta_pq < 0
```

Decrease direction: `alpha_pq < 0` AND `beta_pq > 0`.

Both pixels move in the same direction (both increase or both decrease), because the pair score formula assumes this. Applying one up and one down would violate the mathematical basis of the score.

### Candidate Pruning (top-k)

Evaluating all `n*(n-1)/2` pairs is O(n^2). With 400 valid pixels: 79,800 pairs. Setting `top_k=128` caps evaluation at 8,128 pairs (90% reduction). The pruning heuristic pre-ranks pixels by individual `|alpha| * |beta|` and takes the top-k, then evaluates all pairs among those. This trades a small chance of missing the globally optimal pair for a 90% speedup.

```python
def prune_candidates(alpha, beta, search_space, top_k):
    alpha_masked = alpha * search_space
    beta_masked  = beta  * search_space
    valid = np.where(search_space)[0]
    if valid.size < 2 or top_k is None or valid.size <= top_k:
        return valid
    prelim_scores = np.abs(alpha_masked[valid]) * np.abs(beta_masked[valid])
    idx = np.argsort(-prelim_scores)[:top_k]
    return valid[idx]

def evaluate_pairs(alpha, beta, valid, direction):
    best_p, best_q, best_score = -1, -1, 0.0
    for i in range(valid.size):
        p = valid[i]
        for j in range(i + 1, valid.size):
            q = valid[j]
            a_pq = alpha[p] + alpha[q]
            b_pq = beta[p]  + beta[q]
            if direction == 'increase':
                if a_pq <= 0 or b_pq >= 0:
                    continue
                score = a_pq * abs(b_pq)
            else:
                if a_pq >= 0 or b_pq <= 0:
                    continue
                score = abs(a_pq) * b_pq
            if score > best_score:
                best_score = float(score)
                best_p, best_q = int(p), int(q)
    return best_p, best_q, best_score
```

### Pairwise Attack Loop

```python
config = {'theta': 1.0, 'gamma': 0.15, 'max_iter': 90, 'wrt': 'logits',
          'clip_min': 0.0, 'clip_max': 1.0, 'top_k': 128}

x_adv = x.clone().detach()
search_space = initialize_search_space(x.shape)
pixels_modified = 0
max_pixels = int(config['gamma'] * int(np.prod(x.shape[1:])))

for iteration in range(config['max_iter']):
    if check_target_reached(x_adv, target_class, model):
        break
    if pixels_modified >= max_pixels:
        break

    jacobian = compute_jacobian_matrix(x_adv, model, 10, config['wrt'])
    alpha = extract_target_gradient(jacobian, target_class)
    beta  = extract_other_gradients(jacobian, target_class)

    p_inc, q_inc, score_inc = compute_pairwise_saliency(
        alpha, beta, search_space, 'increase', config['top_k'])
    p_dec, q_dec, score_dec = compute_pairwise_saliency(
        alpha, beta, search_space, 'decrease', config['top_k'])

    if max(score_inc, score_dec) <= 0.0:
        break

    if score_inc >= score_dec:
        p, q, score, increase = p_inc, q_inc, score_inc, True
    else:
        p, q, score, increase = p_dec, q_dec, score_dec, False

    if p < 0 or q < 0:
        break

    x_adv = apply_pair_perturbation(
        x_adv, p, q, config['theta'], increase,
        config['clip_min'], config['clip_max'])

    search_space[p] = False
    search_space[q] = False
    search_space = remove_saturated_pixels(search_space, x_adv, 0.0, 1.0)
    pixels_modified += 2
```

### Pairwise Performance

On the same 10 MNIST samples:
- **Pairwise (theta=1.0):** 90% success (9/10), average 42.8 pixels (5.5% of image), average 22.4 iterations
- **Single-pixel (theta=0.25):** 70% success (7/10), average 73.6 pixels (9.4%), average 74.3 iterations
- Iteration reduction: 69.9%
- The pairwise approach adds ~14% per-iteration overhead (17.1 ms vs 15.0 ms) but needs 70% fewer iterations overall

## Flags and Options

### JSMA Attack Parameters

| Parameter | Role | Typical MNIST value | Effect of increasing |
|-----------|------|---------------------|----------------------|
| `theta` | Step size per modification | 0.25–1.0 | Faster convergence, more visible artifacts |
| `gamma` | Pixel budget fraction | 0.15 | More pixels allowed, higher success rate |
| `max_iter` | Iteration limit | 90–100 | More attempts, slower |
| `top_k` | Pair candidate pruning | 128 | Better pair quality, slower saliency |
| `wrt` | Gradient target | `'logits'` | `'logits'` mandatory for correct behavior |

### Perturbation Norms on MNIST (sample results)

| Norm | Single-pixel (failed) | Pairwise (success) |
|------|-----------------------|--------------------|
| L0 (pixels changed) | 100 | 68 |
| L1 (sum of changes) | ~19.4 | varies |
| L2 (Euclidean) | ~3.5 | varies |
| Linf (max change) | ~0.996 | 1.0 (saturated) |

## Gotchas and Notes

**Shape management is critical.** The Jacobian computation flattens the input to a 1D vector using C-H-W order (channel first, then height, then width). PyTorch's `.view(-1)` uses the same C-contiguous ordering. Mixing flatten orders (e.g., flattening in C-H-W but reading back in H-W-C) silently corrupts pixel locations, producing zero success rate with no error messages.

**Single-pixel vs pairwise terminology.** "Single-pixel" refers to selecting one pixel per iteration, not one total. Both variants are iterative. The pairwise variant selects a pair of pixels per iteration.

**Gradient direction and decrease.** The attack evaluates both increase and decrease directions. A negative alpha gradient does not discard a pixel — it means decreasing that pixel's value will boost the target class. JSMA can "erase" bright pixels as effectively as it can "add" dark ones.

**Saliency score decline.** Scores decrease monotonically during a successful attack as high-value pixels are exhausted. If scores plateau near zero before the target is reached, the attack is stalling — often because the pixel budget is too tight or theta is too small to build up enough cumulative perturbation.

**Memory scaling.** MNIST: 10 classes × 784 pixels = 7,840 values (~31 KB). ImageNet: 1000 × 150,528 = 150M values (~600 MB just for the Jacobian). Chunking (compute gradients for N classes at a time and accumulate on CPU) becomes necessary for large-scale models.

**Architecture limits.** Shallow CNNs like LeNet make individual pixels highly influential. Deep CNNs (ResNet, VGG) distribute decisions across many layers via skip connections and batch normalization, reducing individual pixel influence and degrading JSMA success rates substantially. For complex models, ElasticNet and SparseFool outperform JSMA.

**Batch size must be 1.** The Jacobian computation requires a single image. Passing a batch averages gradients across images, producing meaningless per-sample saliency.

**Computational cost scales with class count.** 10 classes = 10 backward passes per iteration. 1000 classes = 1000 backward passes, making JSMA prohibitively expensive for ImageNet without chunking.

## Challenge: HTB JSMA API

The module challenge provides an API serving:
- `GET /challenge` — returns image (base64 PNG), original label, target class, L0 budget, and max L2 cap
- `GET /weights` — returns PyTorch state dict for a LeNet-5 style `MNISTClassifier`
- `POST /predict` — validates pipeline without returning a flag
- `POST /submit` — accepts adversarial image; returns flag if L0 constraint met, target class predicted, and pixel values in [0, 1]

The server architecture is LeNet-5 style (conv→tanh→avgpool × 2, then FC × 3 with log-softmax output). Images must be submitted in [0, 1] pixel space, not as normalized tensors.

## Related Pages

- [[attack/ai/adversarial_examples]]
- [[attack/ai/elasticnet_attack]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/_overview]]

## Sources

- raw/ai_evasion_sparsity/jsma_fundamentals.md
- raw/ai_evasion_sparsity/jsma_jacobian_and_gradients.md
- raw/ai_evasion_sparsity/jsma_saliency_and_search_spaces.md
- raw/ai_evasion_sparsity/jsma_single-pixel_attack_fundamentals.md
- raw/ai_evasion_sparsity/jsma_single-pixel_attack_loop.md
- raw/ai_evasion_sparsity/jsma_single_pixel_configuration.md
- raw/ai_evasion_sparsity/jsma_single_pixel_batch_evaluation.md
- raw/ai_evasion_sparsity/jsma_pairwise_saliency.md
- raw/ai_evasion_sparsity/jsma_pairwise_attack_loop.md
- raw/ai_evasion_sparsity/jsma_pairwise_batch_evaluation.md
- raw/ai_evasion_sparsity/jsma_pairwise_analysis.md
- raw/ai_evasion_sparsity/jsma_challenge.md
