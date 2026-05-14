---
tags: [attack, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-13
source_count: 4
---

# Iterative FGSM (I-FGSM / BIM / PGD)

Multi-step refinement of [[attack/ai/fgsm]] that re-evaluates the gradient after each small step and projects back to the L∞ budget, achieving substantially higher attack success rates at the same epsilon.

## Overview

I-FGSM (Kurakin et al., 2016), also called the Basic Iterative Method (BIM), extends FGSM by replacing one large step with many small ones. Each iteration takes a step of size α along the gradient sign, then projects back onto the L∞ ball of radius ε around the original image. Because the gradient is re-evaluated at the current adversarial image after every step, the algorithm tracks the curvature of the loss surface rather than trusting a single linear approximation.

At the same ε budget, I-FGSM dramatically improves success rates — typically from ~58% to ~95% on MNIST with ε=0.7. The perturbation budget is never exceeded because projection after each step enforces the per-pixel cap relative to the original image (not the previous iterate).

## Update Rule

Starting from `x⁰ = x`:

```
x^(t+1) = Π_{B∞(x,ε)} ( x^(t) + α · sign(∇_{x^(t)} L(θ, x^(t), y)) )
```

Where the projection clips each pixel's deviation from the original:

```
x^(t+1) = x + clip(x^(t+1) − x, −ε, ε)
x^(t+1) = clip(x^(t+1), x_min, x_max)   # valid input range
```

**Step size:** Default is `α = ε / T` (divide the budget across iterations). Even if all T steps point in the same direction, the total drift is capped at `T · (ε/T) = ε`.

**Relation to PGD:** Projected Gradient Descent (Madry et al., 2017) is the general form. With the sign step, `α = ε/T`, and optional random restarts, I-FGSM coincides with standard L∞ PGD. BIM is the same without random initialization. PGD adds multiple random restarts to escape poor local neighborhoods.

## Implementation

```python
def iterative_fgsm(model, images, labels, epsilon, num_iter,
                   alpha=None, targeted=False, random_start=False):
    NORM_MIN = (0.0 - 0.1307) / 0.3081
    NORM_MAX = (1.0 - 0.1307) / 0.3081
    if alpha is None:
        alpha = epsilon / max(num_iter, 1)

    if random_start:
        delta = torch.empty_like(images).uniform_(-epsilon, epsilon)
        x_adv = torch.clamp(images + delta, NORM_MIN, NORM_MAX)
    else:
        x_adv = images.clone()

    for _ in range(num_iter):
        x_adv = x_adv.detach().requires_grad_(True)
        loss = F.cross_entropy(model(x_adv), labels)
        model.zero_grad(set_to_none=True)
        loss.backward()

        step_dir = -1.0 if targeted else 1.0
        x_adv = x_adv + step_dir * alpha * x_adv.grad.sign()
        # Project onto L∞ ball around original, then clip to valid range
        x_adv = torch.clamp(
            images + (x_adv - images).clamp(-epsilon, epsilon),
            NORM_MIN, NORM_MAX
        )

    return x_adv.detach()
```

**Key projection line:** `(x_adv - images).clamp(-epsilon, epsilon)` measures per-pixel drift from the *original* image and clips any drift exceeding ±ε. This keeps the budget meaning "ε away from the start" throughout all iterations.

## Hyperparameter Trade-offs

| Configuration | Behavior |
|---------------|----------|
| Large α, few T | Fast but may overshoot optimal adversarials |
| Small α, many T | Slower convergence but tracks loss surface curvature more accurately; usually finds stronger adversarials |
| Random start | Adds `δ ∈ [−ε, ε]` before iterating; improves success rate by escaping poor local neighborhoods (~85% → ~92%) |
| No random start (BIM) | Deterministic; faster but may miss adversarials near local optima |

## FGSM vs I-FGSM Comparison

| Metric | FGSM (ε=0.7) | I-FGSM (ε=0.7, T=10) |
|--------|--------------|----------------------|
| Success rate | ~57.8% | ~95.3% |
| Improvement | — | +64.9% relative |
| Budget respected (max L∞) | ε | ε |
| Avg L2 perturbation | lower | slightly higher |

I-FGSM uses the budget more efficiently: it distributes perturbation across pixels in directions that actually cross decision boundaries rather than moving uniformly.

## Complete Metrics at ε=0.8

With ε=0.8 (normalized MNIST), T=10, random_start=True:

```
clean_accuracy:      1.0000
adversarial_accuracy: 0.0000
attack_success_rate: 1.0000
avg_confidence_drop: 0.9739
max_linf_perturbation: 0.8000
```

## Targeted I-FGSM

Same as untargeted but supply the target label in the loss and reverse the step direction. The per-iteration update becomes:

```
x^(t+1) = Π_{B∞(x,ε)} ( x^(t) − α · sign(∇_{x^(t)} L(θ, x^(t), y_t)) )
```

Targeted attacks generally need more iterations or slightly larger ε than untargeted. In testing, a `1→7` targeted flip that requires ε=1.0 with single-step FGSM succeeds at ε=0.8 with I-FGSM (10 iterations).

## Gotchas & Notes

- The projection must be relative to the *original* image, not the previous iterate. Projecting relative to the previous iterate allows unbounded cumulative drift and violates the L∞ budget guarantee.
- Increasing T with `α = ε/T` often raises success rate without increasing `max||δ||∞`. The L2 norm may drop slightly as the budget distributes more efficiently.
- I-FGSM / PGD is the standard attack used in adversarial training research (Madry et al.). Models adversarially trained against PGD are considered robustly trained under the L∞ threat model.
- For black-box settings, craft I-FGSM adversarials on a surrogate model then transfer — success rates drop but remain significant due to transferability.

## Related Pages

- [[attack/ai/fgsm]]
- [[attack/ai/deepfool]]
- [[attack/ai/adversarial_examples]]
- [[tools/attack/art]]

## Sources

- raw/ai_evasion_first_order_attacks/i-fgsm.md
- raw/ai_evasion_first_order_attacks/i-fgsm_implementation.md
- raw/ai_evasion_first_order_attacks/i-fgsm_analysis.md
- raw/ai_evasion_first_order_attacks/i-fgsm_challenge.md
