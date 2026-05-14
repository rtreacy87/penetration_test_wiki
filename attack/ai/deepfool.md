---
tags: [attack, attack/ai]
module: ai_evasion_first_order_attacks
last_updated: 2026-05-13
source_count: 7
---

# DeepFool

Iterative L2-minimal adversarial attack that finds the smallest perturbation needed to cross the nearest decision boundary, by locally linearizing the classifier and taking the orthogonal projection onto the linearized boundary.

## Overview

DeepFool (Moosavi-Dezfooli, Fawzi, Frossard, CVPR 2016) asks a different question than [[attack/ai/fgsm]]: not "given a fixed budget, how much damage can I do?" but "what is the absolute smallest change that fools the model?" It treats adversarial example generation as a geometric problem — find the shortest path from a data point to the nearest decision boundary.

The algorithm uses iterative linearization: approximate the non-linear boundary as a flat hyperplane, take the minimal orthogonal step to that hyperplane, re-evaluate, and repeat. In practice, MNIST samples converge in 2–4 iterations with ~4.9 average L2 norm, and the 100% success rate makes DeepFool the standard tool for computing robustness scores.

## Mathematical Foundation

### Binary case (linear classifier)

For a linear classifier `f(x) = wᵀx + b`, the decision boundary is the hyperplane `f(x) = 0`. The minimal L2 perturbation to reach it from `x₀` is:

```
r* = −(f(x₀) / ||w||₂²) · w
```

This is an orthogonal projection: the step is in the gradient direction `w`, scaled by `|f(x₀)| / ||w||₂²`. The denominator accounts both for normalizing the direction and for how quickly the decision surface can be reached given gradient magnitude. Unlike FGSM's sign step, this preserves relative magnitudes — a pixel with weight 10 changes 10× more than a pixel with weight 1.

### Multi-class extension

For `k` classes with per-class scores `f_k(x)`, at each iteration:

1. For every alternative class `k ≠ k̂(x)`, compute:
   - `w_k = ∇f_k(x_i) − ∇f_k̂(x_i)` — gradient difference (direction that increases class k while decreasing current class)
   - `f_k' = f_k(x_i) − f_k̂(x_i)` — score gap to close
   - `pert_k = |f_k'| / ||w_k||₂` — distance to linearized boundary for class k

2. Select the closest boundary: `l = argmin_k pert_k`

3. Take the minimal step: `r_i = (|f_l'| / ||w_l||₂²) · w_l`

4. Update: `x_{i+1} = x_i + (1 + overshoot) · r_i`

5. Repeat until `k̂(x) ≠ original_label`

The algorithm dynamically selects the nearest boundary each iteration rather than committing to a target class upfront.

## Overshoot Parameter

The step is scaled by `(1 + overshoot)` (typically 0.02). The local linear approximation only holds near the current point — without overshoot, numerical precision can leave the adversarial infinitesimally close to but not across the true non-linear boundary. A 2% overshoot trades a tiny increase in perturbation magnitude for reliable convergence.

## Robustness Metric ρ_adv

DeepFool enables quantitative robustness comparison across models:

```
ρ_adv = (1/|D|) Σ_{x∈D} (||r(x)||₂ / ||x||₂)
```

This is the average relative perturbation size needed to fool the classifier. A model with ρ_adv = 0.02 requires on average 2% perturbations; one with ρ_adv = 0.10 requires 10% — the second model is 5× more robust. Higher ρ_adv = more robust.

## Implementation

```python
def deepfool(image, net, num_classes=10, overshoot=0.02,
             max_iter=50, device='cuda'):
    image = image.to(device)
    net = net.to(device)

    # Initial class ranking
    f_image = net(image).data.cpu().numpy().flatten()
    I = f_image.argsort()[::-1]
    label = I[0]

    pert_image = image.clone()
    r_tot = torch.zeros(image.shape).to(device)
    loop_i = 0

    while loop_i < max_iter:
        x = pert_image.clone().requires_grad_(True)
        fs = net(x)
        k_i = fs.data.cpu().numpy().flatten().argsort()[::-1][0]
        if k_i != label:
            break

        pert = float('inf')
        w = None

        # Find closest boundary among alternative classes
        for k in range(1, num_classes):
            if I[k] == label:
                continue
            # Gradient for candidate class
            if x.grad is not None: x.grad.zero_()
            fs[0, I[k]].backward(retain_graph=True)
            grad_k = x.grad.data.clone()
            # Gradient for original class
            if x.grad is not None: x.grad.zero_()
            fs[0, label].backward(retain_graph=True)
            grad_label = x.grad.data.clone()

            w_k = grad_k - grad_label
            f_k = (fs[0, I[k]] - fs[0, label]).data.cpu().numpy()
            pert_k = abs(f_k) / (torch.norm(w_k.flatten()) + 1e-10)

            if pert_k < pert:
                pert = pert_k
                w = w_k

        # Minimal step with overshoot
        r_i = (pert + 1e-4) * w / (torch.norm(w.flatten()) + 1e-10)
        r_tot = r_tot + r_i
        pert_image = image + (1 + overshoot) * r_tot
        loop_i += 1

    return r_tot, loop_i, label, k_i, pert_image
```

## Comparison with FGSM and I-FGSM

| Property | FGSM | I-FGSM | DeepFool |
|----------|------|---------|----------|
| Constraint | L∞ fixed budget | L∞ fixed budget | L2 minimal |
| Steps | 1 | T (configurable) | Until boundary crossed |
| Sign step | Yes (discards magnitude) | Yes | No (preserves magnitude) |
| Target class | No (or fixed) | No (or fixed) | Auto-selects nearest boundary |
| Perturbation | Not minimal | Not minimal | Approximately minimal |
| Use case | Speed, baseline | Stronger attacks, adv. training | Robustness measurement, minimal perturbations |

Key difference: I-FGSM takes uniform steps across all pixels using the sign; DeepFool takes geometrically weighted steps where pixels with stronger gradient components receive proportionally larger perturbations.

## Batch Results on MNIST

Processing 20 samples individually:
```
Attack Success Rate: 20/20 (100.0%)
Average L2 norm: 4.87
L2 range: [0.60, 7.72]
Average iterations: 3.0
```

The 13× range in L2 norms reveals how decision boundary geometry varies across the input space. '3→5' requires only L2=0.60 (structurally similar); '7→2' requires L2=7.72 (significant structural differences). Common misclassifications on MNIST: `9→4`, `7→2`, `2→6` (geometrically adjacent classes in feature space).

## L∞ Variant

DeepFool generalizes beyond L2. For L∞-DeepFool, the boundary selection changes:

```
l̂ = argmin_k |f_k'| / ||w_k||₁
r_i = (|f_l'| / ||w_l||₁) · sign(w_l)
```

The denominator switches to L1 norm and the direction uses element-wise sign, distributing perturbation uniformly across pixels under an L∞ constraint.

## Universal Adversarial Perturbations

DeepFool's minimal perturbation principle inspired universal adversarial perturbations: a single δ that fools a model on *most* inputs from a dataset. The algorithm iteratively applies DeepFool to different training examples, accumulating perturbations with magnitude constraints. The existence of universal perturbations demonstrates systematic rather than per-input model vulnerabilities.

## Gotchas & Notes

- `num_classes` limits the search to top-k classes per iteration. On ImageNet (1000 classes), setting `num_classes=10` reduces gradient computations by 99% while maintaining effectiveness. For MNIST (10 classes), use all 10.
- `retain_graph=True` is required on all but the last backward call in the candidate class loop; without it PyTorch frees the computation graph after the first call.
- DeepFool does not enforce a domain validity constraint (e.g., [0,1] pixel bounds) during iteration. In practice the returned `pert_image` may need clamping before use as a valid input.
- High confidence on a clean example does not guarantee large ρ_adv. Deep networks routinely show high confidence on samples that sit very close to decision boundaries — this is one of the core anomalies adversarial examples expose.
- DeepFool adversarials concentrate perturbations along class-discriminative features (digit strokes, edges). The L2/L∞ ratio is typically ~6× larger than a uniform distribution would predict, confirming targeted rather than scattered modification.

## Related Pages

- [[attack/ai/fgsm]]
- [[attack/ai/i_fgsm]]
- [[attack/ai/adversarial_examples]]
- [[tools/attack/art]]
- [[labs/htb/ai_evasion_first_order_attacks/deepfool_challenge]]

## Sources

- raw/ai_evasion_first_order_attacks/deepfool_theory_and_formulation.md
- raw/ai_evasion_first_order_attacks/deepfool_and_the_quest_for_minimality.md
- raw/ai_evasion_first_order_attacks/deepfool_and_training_the_target_model.md
- raw/ai_evasion_first_order_attacks/deepfool_implementation.md
- raw/ai_evasion_first_order_attacks/deepfool_perturbation_analysis_and_metrics.md
- raw/ai_evasion_first_order_attacks/demonstrating_deepfool.md
- raw/ai_evasion_first_order_attacks/batch_attack_generation.md
