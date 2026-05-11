---
tags: [attack, attack/ai]
module: ai_evasion_sparsity
last_updated: 2026-05-10
source_count: 16
---

# ElasticNet (EAD) Adversarial Attack

The Elastic-net Attacks to Deep Neural Networks (EAD) combine L1 and L2 regularization to produce sparse adversarial perturbations, optimized with FISTA and automatically tuned via binary search over the adversarial pressure constant.

## Overview

EAD (Chen et al., 2018, "EAD: Elastic-Net Attacks to Deep Neural Networks via Adversarial Examples") extends the Carlini & Wagner (C&W) attack framework with an L1 regularization term that drives perturbations toward sparsity. While pure L2 attacks (like C&W) spread tiny changes across every pixel, ElasticNet concentrates modifications on a smaller set of high-impact pixels. The key insight is that L1's non-differentiable corner at zero causes optimization to zero out many coordinates exactly rather than leaving them at tiny non-zero values, while L2 provides the smooth gradients needed for efficient gradient-based optimization.

The result is perturbations that are simultaneously sparse (few pixels change) and calibrated in magnitude — different from JSMA which trades magnitude control for hard L0 counting, and different from C&W which trades sparsity for smooth L2 perturbations.

## Key Concepts and Techniques

### The Sparsity-Smoothness Trade-off

Three norm types encode three different philosophies:

**L2 (C&W baseline):** Smooth, dense perturbations. Every pixel contributes slightly. Good gradient properties throughout optimization. No sparsity.

**L1:** Sparse perturbations. Its diamond-shaped constraint set has sharp corners on coordinate axes, so the optimizer tends to place solutions at corners where many coordinates are exactly zero. Non-differentiable at zero — naive gradient descent hovers near zero but rarely achieves exact sparsity.

**ElasticNet (L1 + L2):** Combines both. L2 provides smooth gradients for stable optimization; L1 induces sparsity. A concrete example:
- Case A: 100 pixels changed by 0.10 each. Squared L2 = 1.00, L1 = 10.0
- Case B: 10 pixels changed by 0.316 each. Squared L2 ≈ 1.00, L1 ≈ 3.16

Both cases have similar L2 — the smoothness term treats them as roughly equivalent. The L1 term clearly prefers Case B (sparser). Higher beta pushes the optimizer toward Case B; lower beta behaves like pure L2 and tolerates Case A.

### Complete Objective Function

The smooth component that FISTA differentiates (L1 is handled separately by the proximal operator):

```
L_total = c * f(x', y) + ||x' - x||_2^2
```

Where `f(x', y)` is the adversarial loss (margin-based, from C&W), `c` is the trade-off constant tuned by binary search, and the L2 term provides a restoring force pulling perturbations back toward the original image.

The L1 component enters through FISTA's proximal step:
```
h(x') = beta * ||x' - x||_1
```

Full elastic-net distance for analysis/reporting:
```
elastic_dist = ||x' - x||_2^2 + beta * ||x' - x||_1
```

### C&W Adversarial Loss

The margin-based loss measures how far the attack is from misclassification. For untargeted attacks (force any wrong class):

```
f(x', y) = max(Z_y(x') - max_{j != y} Z_j(x') + kappa, 0)
```

Where `Z_y` is the true class logit, the max term is the strongest competitor's logit, and `kappa` is a confidence margin. When the true class still leads by more than kappa, the loss is positive and provides gradient signal to reduce that gap. Once any competitor exceeds the true class by kappa, the loss saturates at 0 — optimization shifts entirely to minimizing distortion.

For targeted attacks (force class t), the roles flip:
```
f(x', y) = max(max_{j != t} Z_j(x') - Z_t(x') + kappa, 0)
```

With `kappa=0`, any misclassification succeeds. With `kappa>0`, the attacker demands a more decisive margin — producing more robust adversarial examples at the cost of larger perturbations.

```python
def compute_adversarial_loss(logits, labels_onehot, confidence, targeted=False):
    # Extract true class score via one-hot mask
    real  = torch.sum(labels_onehot * logits, dim=1)
    # Maximum competitor score (true class masked out with -10000)
    other = torch.max((1 - labels_onehot) * logits - labels_onehot * 10000, dim=1)[0]

    if targeted:
        loss = torch.clamp(other - real + confidence, min=0)
    else:
        loss = torch.clamp(real - other + confidence, min=0)
    return loss
```

## FISTA Optimization

### Why Standard Gradient Descent Fails for L1

The L1 norm `||x||_1 = sum |x_i|` has a non-differentiable kink at zero for every coordinate. Applying gradient descent to minimize L1 produces values that get smaller and smaller but never reach exactly zero — pseudo-sparsity. FISTA (Fast Iterative Shrinkage-Thresholding Algorithm) handles this by decomposing the problem into:
- **Smooth part (f):** adversarial loss + L2 distance — differentiated normally via backprop.
- **Non-smooth part (h):** L1 penalty — handled by a proximal operator instead of a gradient.

### Soft Thresholding: The L1 Proximal Operator

The proximal operator for `h(x) = lambda * ||x||_1` is element-wise soft thresholding:

```
S_lambda(z)_i = z_i - lambda  if z_i > lambda
              = 0              if |z_i| <= lambda
              = z_i + lambda  if z_i < -lambda
```

This operation has an elegant interpretation: values with magnitude below the threshold are zeroed out exactly (true sparsity), while larger values are shrunk by exactly lambda toward zero. It solves the one-dimensional optimization `minimize 0.5*(x-z)^2 + lambda*|x|` in closed form.

Example with `lambda = 0.1`:
- `z = 0.12` → `0.02` (survives, shrunk)
- `z = 0.08` → `0.00` (eliminated — exact zero)
- `z = -0.25` → `-0.15` (survives negative side, shrunk toward zero)

The full operator applies this element-wise, enabling efficient parallel computation. This is what ElasticNet uses to enforce sparsity in perturbations at each FISTA iteration.

```python
def apply_shrinkage_thresholding(y, original_images, threshold, clip_min=0.0, clip_max=1.0):
    diff = y - original_images
    shrink_positive = torch.clamp(y - threshold, min=clip_min, max=clip_max)
    shrink_negative = torch.clamp(y + threshold, min=clip_min, max=clip_max)
    cond_positive = (diff > threshold).float()
    cond_zero     = (torch.abs(diff) <= threshold).float()
    cond_negative = (diff < -threshold).float()
    result = (cond_positive * shrink_positive +
              cond_zero     * original_images  +
              cond_negative * shrink_negative)
    return result
```

A beta=0.1 threshold on a 25-element test pattern eliminated 13 of 22 non-zero elements (64% sparsity achieved), converting weak perturbations to exactly zero.

### Nesterov Momentum

FISTA accelerates proximal gradient descent using Nesterov momentum. Instead of computing gradients at the current solution `x(k)`, FISTA computes them at an extrapolated "look-ahead" point `y(k)` that anticipates the direction of progress:

```
x(k+1) = prox(y(k) - eta * grad_f(y(k)))
t(k+1) = k / (k + 3)   [simplified Nesterov schedule]
y(k+1) = x(k+1) + t(k+1) * (x(k+1) - x(k))
```

The momentum coefficient `t_k = k / (k+3)` grows from cautious (0.25 at iteration 1) to aggressive (0.97 at iteration 100). This adaptive schedule eliminates manual momentum tuning: early iterations explore without commitment; later iterations exploit consistent descent directions. FISTA achieves O(1/k^2) convergence versus O(1/k) for standard proximal gradient descent.

```python
def compute_fista_momentum(iteration):
    return iteration / (iteration + 3.0)
```

Progression: iter 1 → 0.25, iter 10 → 0.77, iter 100 → 0.97, iter 1000 → 0.997.

### Complete FISTA Step

```python
def fista_step(adv_images, y_momentum, original_images, labels_onehot,
               const, model, beta, learning_rate, confidence, iteration,
               targeted=False, clip_min=0.0, clip_max=1.0):
    # Break old graph, start fresh for this iteration
    y_momentum = y_momentum.detach().requires_grad_(True)

    # Evaluate loss at look-ahead point (not at current solution)
    total_loss, adversarial_loss, distances = compute_total_loss(
        y_momentum, original_images, labels_onehot, const, model,
        beta, confidence, targeted)

    # Backprop to get gradient of smooth terms
    total_loss_summed = total_loss.sum()
    total_loss_summed.backward()
    grad = y_momentum.grad

    # Gradient step on smooth terms
    y_new = y_momentum - learning_rate * grad

    # Proximal step: soft thresholding enforces L1 sparsity
    adv_new = apply_shrinkage_thresholding(
        y_new, original_images, learning_rate * beta, clip_min, clip_max)

    # Nesterov momentum extrapolation for next iteration
    momentum_coef = compute_fista_momentum(iteration)
    y_new_momentum = adv_new + momentum_coef * (adv_new - adv_images)

    return adv_new, y_new_momentum, total_loss_summed.item(), distances
```

Key design: loss is evaluated at `y_momentum` (the look-ahead point), not at `adv_images` (current solution). The gradient step and shrinkage produce the new solution `adv_new`. Then the momentum formula extrapolates `y_new_momentum` for the next iteration's look-ahead. Detach-then-require-grad breaks the previous iteration's computation graph, preventing gradient accumulation across FISTA steps.

## Binary Search for Constant c

The trade-off constant `c` balances adversarial pressure against distortion minimization. Too small: the model won't misclassify. Too large: perturbations become unnecessarily large. Binary search finds the minimal `c` sufficient for each example automatically.

**Mechanism:**
- Start: `lower=0`, `upper=1e10`, `const=0.001`
- Success (model fooled): lower upper bound to current `c`, bisect interval downward
- Failure (model not fooled): raise lower bound, bisect upward if bounded, otherwise multiply by 10 (exponential growth to find right scale)
- After each binary search step, FISTA restarts from original images with updated `c` values

```python
def update_binary_search_bounds(lower_bound, upper_bound, const, success_mask):
    for i in range(len(success_mask)):
        if success_mask[i]:
            upper_bound[i] = min(upper_bound[i], const[i])
            if upper_bound[i] < 1e10:
                const[i] = (lower_bound[i] + upper_bound[i]) / 2
        else:
            lower_bound[i] = max(lower_bound[i], const[i])
            if upper_bound[i] < 1e10:
                const[i] = (lower_bound[i] + upper_bound[i]) / 2
            else:
                const[i] *= 10  # exponential increase until first success
    return lower_bound, upper_bound, const
```

Binary search is **per-example**: easy examples near the decision boundary need small `c` values; resistant examples need large ones. Processing them with the same fixed `c` would over-perturb easy cases or under-perturb hard ones. Per-example bounds handle this automatically.

## Complete Attack Execution

### Attack Configuration

```python
config = {
    "beta":               0.01,   # L1/L2 trade-off (higher = sparser)
    "confidence":         0,      # kappa margin (0 = any misclassification counts)
    "learning_rate":      0.01,   # FISTA step size
    "max_iterations":     1000,   # FISTA iterations per binary search step
    "binary_search_steps": 5,     # outer search iterations
    "initial_const":      0.001,  # starting c value
    "clip_min":           0.0,
    "clip_max":           1.0,
}
```

### Nested Loop Structure

```python
for binary_step in range(config["binary_search_steps"]):
    # Fresh initialization at each binary search step
    adv_images = original_images.clone().detach()
    y_momentum = adv_images.clone()

    for iteration in range(config["max_iterations"]):
        adv_images, y_momentum, loss, distances = fista_step(
            adv_images, y_momentum, original_images, labels_onehot,
            const, model, config["beta"], config["learning_rate"],
            config["confidence"], iteration, targeted=False,
            clip_min=config["clip_min"], clip_max=config["clip_max"])

        success_mask = check_attack_success(adv_images, attack_targets, model)

        # Track best (lowest L2) successful adversarial per example
        l1_dist, l2_dist, elastic_dist = compute_distances(
            adv_images, original_images, config["beta"])
        for i in range(batch_size):
            if success_mask[i] and l2_dist[i] < best_l2[i]:
                best_adv[i] = adv_images[i]
                best_l2[i]  = l2_dist[i]

    lower_bound, upper_bound, const = update_binary_search_bounds(
        lower_bound, upper_bound, const, success_mask)
```

**Fresh restart at each binary search step is mandatory.** Without it, perturbations optimized for `c=0.001` contaminate the optimization for `c=0.01`, producing unreliable results. Each constant value gets a fair, independent evaluation.

## Distance Metrics

```python
def compute_distances(adv_images, original_images, beta):
    l1_dist      = torch.sum(torch.abs(adv_images - original_images), dim=(1,2,3))
    l2_dist      = torch.sum((adv_images - original_images)**2,       dim=(1,2,3))
    elastic_dist = l2_dist + beta * l1_dist
    return l1_dist, l2_dist, elastic_dist
```

Note: the implementation uses **squared** L2 (sum of squared differences), not the Euclidean norm. This simplifies the gradient to `2*(x' - x)` and maintains strict convexity. Reported as "squared L2 distance" in results.

`dim=(1,2,3)` collapses channel, height, and width dimensions while preserving the batch dimension — producing one distance value per image for per-example binary search adaptation.

## Sparsity Analysis

ElasticNet with `beta=0.01` on 20 MNIST examples achieved:
- 100% success rate (20/20)
- Mean sparsity: 52.6% (about half the pixels unchanged)
- Mean squared L2: 4.18
- Sparsity range: ~35%–75% across examples

Perturbations concentrate along digit edges and stroke boundaries — the regions where gradient magnitudes are largest. Background pixels receive near-zero gradient signals, triggering soft thresholding elimination. This spatial concentration is automatic, not manually imposed.

The `beta` parameter controls sparsity aggressiveness:
- `beta=0.001–0.005`: nearly dense, behaves like pure L2
- `beta=0.01`: moderate sparsity (default, ~52%)
- `beta=0.05–0.1`: aggressive sparsity, may reduce success rate 5–10% on hard examples

## Flags and Options

### Hyperparameter Reference

| Parameter | Role | Typical value | Sensitivity |
|-----------|------|---------------|-------------|
| `beta` | L1 weight / sparsity control | 0.01 | Moderate: doubling adds ~10–15% sparsity |
| `confidence` (`kappa`) | Required misclassification margin | 0 | Higher = more robust AEs, larger perturbations |
| `learning_rate` | FISTA step size | 0.01 | **High:** >0.05 causes divergence, <0.001 needs 2000+ iters |
| `max_iterations` | FISTA iterations per binary search step | 1000 | More = better convergence, proportional compute |
| `binary_search_steps` | Outer loop iterations | 5–9 | 5=faster (+15% distortion), 9=minimal distortion (+80% compute) |
| `initial_const` | Starting c value | 0.001 | Low start allows binary search to find minimum |

### Runtime (MNIST, 20 samples, GPU)

| Setting | Time estimate |
|---------|---------------|
| 5 binary search steps × 1000 FISTA iters | 2–3 minutes (RTX 3090) |
| CPU equivalent | 30–60 minutes |
| Batch size 20 | ~400 MB GPU memory |
| Batch size 50 | ~1 GB GPU memory |

### Comparison: EAD vs C&W vs JSMA

| Property | JSMA | C&W (L2) | ElasticNet (EAD) |
|----------|------|----------|-----------------|
| Primary norm | L0 (count) | L2 (energy) | L1+L2 (elastic) |
| Optimization | Greedy saliency | Adam gradient descent | FISTA + proximal |
| Sparsity | Hard (exact pixel budget) | None | Soft (L1 penalty) |
| Perturbation character | Saturated pixels, sparse | Smooth, dense | Sparse + smooth |
| Per-example tuning | No | Optional (C&W binary search) | Yes (binary search) |
| Computational cost | O(classes * iters) | Moderate | High (nested loops) |
| Architecture dependence | High (shallow only) | Low | Low |

## Gotchas and Notes

**L1 gradient does not exist at zero.** Never apply standard gradient descent directly to the L1 term. FISTA's proximal operator handles this correctly — the soft thresholding step replaces what would be an undefined gradient operation.

**Detach before re-enabling gradients in FISTA.** Each FISTA iteration must call `.detach().requires_grad_(True)` on the momentum point to break the previous iteration's computation graph. Without detach, PyTorch accumulates an ever-growing graph causing memory leaks and incorrect gradients.

**Fresh FISTA initialization per binary search step.** Reusing adversarial images across binary search iterations with different `c` values contaminates the optimization. Always restart from `original_images.clone()`.

**Squared vs Euclidean L2.** The implementation uses squared L2 (`||delta||_2^2`) for its simpler gradient `2*delta`. Reported metrics should specify which form is used, as values differ significantly.

**Batch-level vs per-example const.** ElasticNet adapts `c` independently for each example. This is key to handling heterogeneous robustness: easy examples converge with `c ≈ 0.001`; hard examples may need `c ≈ 10`.

**Beta=0 collapses to C&W.** Setting `beta=0` eliminates the L1 term, making ElasticNet identical to C&W L2 attack (minus some minor implementation differences). This is a useful baseline for comparison.

**The confidence margin kappa affects transferability.** With `kappa=0`, the adversarial example crosses the decision boundary by just enough to misclassify. With `kappa=5`, the attack must push 5 logit units past the boundary, producing examples that remain adversarial under small perturbations or preprocessing — more transfer-resistant.

**Learning rate sensitivity is high.** `learning_rate > 0.05` typically causes divergence on MNIST-scale problems, producing oscillating perturbations instead of convergence. `learning_rate < 0.001` converges slowly, requiring 2000–5000 iterations for the same quality. `0.01` is the sweet spot for MNIST.

## Challenge: HTB ElasticNet API

The module challenge provides an API with stricter multi-norm constraints than JSMA:
- `GET /challenge` — returns image (base64 PNG), label, beta, elastic_max, l2_max, l1_max
- `GET /weights` — returns state dict for a `SimpleClassifier` (2 conv + 2 FC, ReLU, dropout)
- `POST /predict` — validates pipeline
- `POST /submit` — checks all three distance constraints simultaneously (elastic-net, L2, L1) and misclassification; returns flag on success

The `SimpleClassifier` architecture uses ReLU activations, max pooling, and dropout (unlike the LeNet-5 with tanh in the JSMA challenge). It returns raw logits, not log-softmax — gradient computation and loss functions work the same way, but the output interpretation differs slightly.

Error messages from the submission endpoint identify exactly which constraint was violated, enabling systematic tightening of the attack.

## Related Pages

- [[attack/ai/adversarial_examples]]
- [[attack/ai/jsma]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/_overview]]

## Sources

- raw/ai_evasion_sparsity/elasticnet.md
- raw/ai_evasion_sparsity/elasticnet_environment_setup.md
- raw/ai_evasion_sparsity/elasticnet_adversarial_loss.md
- raw/ai_evasion_sparsity/elasticnet_loss_gradients_and_optimization.md
- raw/ai_evasion_sparsity/elasticnet_proximal_operators_amd_fista_framework.md
- raw/ai_evasion_sparsity/elasticnet_implementing_fista_components.md
- raw/ai_evasion_sparsity/elasticnet_complete_fista_iteration_and_binary_search.md
- raw/ai_evasion_sparsity/elasticnet_distance_metrics.md
- raw/ai_evasion_sparsity/elasticnet_sparsity_analysis.md
- raw/ai_evasion_sparsity/elasticnet_attack_execution_and_performance.md
- raw/ai_evasion_sparsity/elasticnet_attack_challenge.md
- raw/ai_evasion_sparsity/elasticnet_attack_visualizations.md
- raw/ai_evasion_sparsity/intrduction_to_sparsity_evasion_attacks.md
