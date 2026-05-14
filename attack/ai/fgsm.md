---
tags: [attack, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-13
source_count: 8
---

# Fast Gradient Sign Method (FGSM)

Single-step L∞-bounded adversarial attack that uses the sign of the input gradient to produce adversarial examples in one forward-backward pass.

## Overview

FGSM (Goodfellow et al., 2014) is the foundational gradient-based evasion attack. Given a trained model, it computes the gradient of the loss with respect to the input image, takes only the sign of each gradient component, and adds a scaled version of that sign pattern to the original image. The result is an adversarial example that crosses the model's decision boundary while keeping every pixel within an L∞ budget of ε.

The attack is intentionally coarse: the sign operation discards magnitude information entirely, treating a gradient component of 0.001 identically to one of 100. This makes it fast and simple while remaining effective as a baseline. FGSM underpins a large body of adversarial training literature and is the standard first test for measuring model robustness.

## The Update Rule

For untargeted FGSM:

```
x_adv = x + ε · sign(∇_x L(θ, x, y))
```

For targeted FGSM (force prediction toward target class y_t):

```
x_adv = x − ε · sign(∇_x L(θ, x, y_t))
```

The sign step ensures every pixel in the perturbation has magnitude exactly ε (or zero when the gradient is exactly zero, which is rare). The perturbation is then clipped so the result stays within the valid input range.

## Key Concepts

**Epsilon (ε):** Controls the L∞ budget — the maximum allowed change per pixel. Small ε (e.g., 8/255 ≈ 0.031 in pixel space) is barely perceptible. Large ε degrades visual quality but raises attack success rate.

**Epsilon space:** When a model normalizes inputs internally (`x_norm = (x - μ) / σ`), ε must be expressed in the same space as the gradient. An ε of 0.8 in normalized MNIST space corresponds to ~0.25 in [0,1] pixel space (`ε_pixel = σ · ε_norm`).

**Normalization and gradient quality:** Models trained on normalized inputs produce sharper decision boundaries and more informative input gradients, making them simultaneously more accurate and more vulnerable to FGSM. The same gradient quality that enables fast training also exposes the model to efficient evasion.

**Single-step limitation:** FGSM approximates the loss surface as locally linear. Where the surface curves significantly, the single step may miss the decision boundary or land in a poor adversarial region. Iterative variants ([[attack/ai/i_fgsm]]) address this by re-evaluating the gradient after each step.

## Implementation

```python
def fgsm_attack(model, images, labels, epsilon, targeted=False):
    # Assumes images are already normalized; epsilon in normalized space
    NORM_MIN = (0.0 - 0.1307) / 0.3081   # MNIST bounds
    NORM_MAX = (1.0 - 0.1307) / 0.3081

    x_req = images.clone().detach().requires_grad_(True)
    logits = model(x_req)
    loss = F.cross_entropy(logits, labels)
    model.zero_grad(set_to_none=True)
    loss.backward()

    step_dir = -1.0 if targeted else 1.0
    x_adv = images + step_dir * epsilon * x_req.grad.sign()
    return torch.clamp(x_adv, NORM_MIN, NORM_MAX).detach()
```

### Pixel-space variant

When inputs arrive in [0,1] but the model expects normalized inputs, gradients are in normalized space and must be converted before applying the pixel-space ε:

```
grad_img = grad_normalized / σ   # undo normalization scaling
x_adv = clamp(x + ε · sign(grad_img), 0, 1)
```

## Evaluation Metrics

| Metric | What it measures |
|--------|-----------------|
| Attack success rate | Fraction of originally-correct samples flipped |
| Adversarial accuracy | Model accuracy on adversarial batch |
| Avg confidence drop | Reduction in model's probability for true class |
| Avg L2 perturbation | Euclidean distance from clean to adversarial |
| Max L∞ perturbation | Should equal ε (confirms budget is respected) |

With ε = 0.8 in MNIST normalized space (~0.25 pixel space), a well-trained SimpleCNN typically shows:
- Attack success rate: ~68–72%
- Average confidence drop: ~59 percentage points
- Max L∞: 0.8000 (budget respected)

Compare to [[attack/ai/i_fgsm]] at the same budget: ~95–100% success rate.

## Targeted FGSM

Targeted attacks require larger ε or more iterations to succeed because they must steer toward a specific class rather than simply escape the current one. A `1→7` targeted flip on MNIST typically succeeds at ε=0.8 in normalized space where ε=0.5 fails.

```python
target_label = torch.tensor([7], device=device)
x_adv = fgsm_attack(model, image, target_label, epsilon=0.8, targeted=True)
```

## Flags & Options

| Parameter | Typical value | Effect |
|-----------|---------------|--------|
| epsilon | 0.031 (8/255 pixel) | Imperceptible L∞ budget |
| epsilon | 0.8 (MNIST normalized) | ≈0.25 in pixel space, clearly effective |
| targeted | False | Any misclassification |
| targeted | True | Specific class, needs larger ε |

## Gotchas & Notes

- Always clamp adversarial outputs to the valid input range after the update step. Unnormalized pixel space clips to [0,1]; normalized MNIST clips to [-0.424, 2.821].
- The sign operation means all non-zero gradient components contribute a perturbation of exactly ε regardless of gradient magnitude. This can result in effective but not minimal perturbations — [[attack/ai/deepfool]] finds smaller ones.
- FGSM adversarials transfer to other models (even different architectures) because gradients from similar training distributions point in similar directions. This enables black-box attacks via surrogate models.
- FGSM is used in adversarial training: training on FGSM-perturbed examples makes models more robust but typically reduces clean accuracy slightly.
- Both OWASP ML Security Top 10 (ML01:2023 input manipulation) and Google's SAIF framework cite gradient-based evasion attacks as top ML threats.

## Related Pages

- [[attack/ai/i_fgsm]]
- [[attack/ai/deepfool]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/jsma]]
- [[attack/ai/elasticnet_attack]]
- [[tools/attack/art]]
- [[labs/htb/ai_evasion_first_order_attacks/fgsm_challenge]]

## Sources

- raw/ai_evasion_first_order_attacks/introduction_to_first_order_attacks.md
- raw/ai_evasion_first_order_attacks/fgsm.md
- raw/ai_evasion_first_order_attacks/understanding_norms.md
- raw/ai_evasion_first_order_attacks/fgsm_setup.md
- raw/ai_evasion_first_order_attacks/fgsm_core_implementation.md
- raw/ai_evasion_first_order_attacks/fgsm_normalization.md
- raw/ai_evasion_first_order_attacks/fgsm_evaluation_metrics.md
- raw/ai_evasion_first_order_attacks/targeted_fgsm.md
