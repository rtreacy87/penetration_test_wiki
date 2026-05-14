---
tags: [attack, attack/ai, concept]
module: ai_evasion_sparsity
last_updated: 2026-05-10
source_count: 2
---

# Adversarial Examples and Evasion Attacks Against ML Models

A primer on the threat model, mathematical constraints, and attack families used to craft inputs that fool trained machine learning classifiers at inference time.

## Overview

Adversarial examples are inputs that have been deliberately modified — often imperceptibly — to cause a machine learning model to produce an incorrect output. At inference time, an attacker with access to model gradients or query access can perturb inputs in ways the model misclassifies while human observers see nothing unusual. Understanding adversarial examples is foundational to AI security work: they underpin red-team evaluations of ML-based detectors, classifiers, and filtering systems.

## Threat Model

### Attacker Position

Two main positions exist on the white-box/black-box spectrum:

**White-box:** The attacker can run the model and compute gradients through it. This covers situations where source code and weights are available — common in research environments, supply-chain scenarios, or after a model theft. White-box access allows exact gradient computation, making attacks significantly more efficient.

**Black-box:** The attacker can only observe outputs (class labels, confidence scores, or probability distributions) in response to queries. Gradients must be estimated via finite differences, or attacks must be transferred from a surrogate model trained on similar data. Black-box attacks are slower but realistic against production API endpoints.

Most of the techniques in this module assume white-box access because they require computing exact Jacobians or loss gradients. Black-box variants exist (query-based estimation, model transfer) but are not covered in the source material.

### Validity Constraints

Any adversarial example must remain a valid input in the original domain. For image classifiers this means pixel values must stay in `[0, 1]`. If the model applies normalization internally (`x_hat = (x - mu) / sigma`), gradients still propagate through the normalization by the chain rule, so reasoning in pixel space remains correct.

## Perturbation Norms and Budgets

The key design decision in any evasion attack is what constraint to place on the perturbation `delta = x_adv - x`. Different norms encode different threat models and detectability assumptions.

### L-infinity (Linf)

Every pixel is allowed to move by at most epsilon. FGSM and PGD use this norm. Perturbations spread uniformly across all pixels — visually they look like faint noise superimposed on the image. Easy to detect by checking maximum absolute pixel shift.

### L2

Minimizes the Euclidean distance in pixel space. DeepFool uses this. Perturbations are smooth and distributed; every pixel contributes something. The total energy of the perturbation is bounded but individual pixels can move more than Linf allows if others move less.

### L1

Minimizes the sum of absolute pixel changes. L1's key geometric property is that its unit ball has sharp corners on coordinate axes, which causes optimization to naturally zero out many coordinates. The result is sparse perturbations: most pixels are untouched while a smaller number change by larger amounts. L1 is non-differentiable at zero, making optimization harder than L2 — FISTA's proximal operators are the standard solution.

### L0

Counts the number of pixels that differ at all:

```
||x_adv - x||_0 = |{i | (x_adv)_i != x_i}|
```

This is the sparsity constraint in its purest form. L0 is non-convex and non-differentiable, so attacks that minimize it directly use greedy strategies (JSMA) rather than gradient descent. L0 attacks align with real attacker constraints: flipping a few bits in a binary, changing a handful of pixels within rendering limits, or modifying a small number of tokens in text.

### Norm Comparison Table

| Norm | Perturbation character | Typical attack | Detectability |
|------|------------------------|----------------|---------------|
| Linf | Dense, bounded magnitude | FGSM, PGD | Detectable via max-pixel check |
| L2 | Dense, smooth | DeepFool, C&W | Hard to isolate |
| L1 | Sparse, larger individual changes | ElasticNet (EAD) | Detectable as local edits |
| L0 | Minimal pixel count, any magnitude | JSMA | Visible as dots/strokes |

## Two Paths to Sparsity

The source material focuses on two attacks that specifically pursue sparse perturbations, each with a distinct strategy:

**JSMA (Jacobian-based Saliency Map Attack)** enforces an explicit L0 budget by selecting one or two pixels per iteration using a saliency map derived from the input Jacobian. Each selected pixel can jump dramatically (black to white or vice versa) because the constraint is on count, not magnitude.

**ElasticNet (EAD)** promotes sparsity indirectly by adding an L1 penalty to the optimization objective alongside L2. The full objective is:

```
minimize: c * f(x') + ||x' - x||_2^2 + beta * ||x' - x||_1
subject to: x' in [0, 1]
```

The L1 term's non-differentiability at zero causes optimization to zero out low-impact pixels naturally, producing sparse perturbations without a hard pixel count limit. Beta controls how aggressively sparsity is enforced.

## Why Sparsity Attacks Matter

Sparse perturbations align with real attack constraints and have security-relevant properties:

- An attacker modifying a network packet or binary can change only a limited number of bytes without detection.
- Sparse edits may evade anomaly detectors tuned to global noise statistics, since those detectors look for widespread coordinated change.
- Sparse attacks reveal which features the model considers most decisive — a useful interpretability signal for defenders.
- For defenders, reproducing JSMA and EAD establishes baselines for L0 and L1-induced sparsity, exposing different failure modes than Linf or L2 attacks.

## Targeted vs Untargeted Attacks

**Untargeted:** Force any misclassification. The attack succeeds if the model predicts any class other than the true one.

**Targeted:** Force classification as a specific target class. Much harder — requires not just crossing one decision boundary but steering toward a specific region of the output space. JSMA is inherently targeted; ElasticNet supports both modes through its loss function.

## First-Order vs Sparsity-Focused Methods

The first-order methods (FGSM, DeepFool) use gradient sign or gradient direction to find perturbation directions. They penalize perturbation size but allow all features to move. Sparsity attacks shift the constraint from "how small" to "how few features change":

- FGSM: single step, Linf constraint, all pixels move by at most epsilon.
- DeepFool: iterative, L2 constraint, smallest perturbation to nearest boundary.
- JSMA: iterative, L0 constraint, one or two pixels per step, large magnitude allowed.
- ElasticNet: iterative with FISTA, L1+L2 constraint, soft sparsity through proximal operators.

## Gotchas and Notes

**Logits vs probabilities for gradient computation:** When computing gradients for saliency scoring, always differentiate through logits (pre-softmax), not probabilities. Post-softmax, probabilities must sum to one, which forces the gradient of competitors to always equal the negative gradient of the target. This collapses JSMA's saliency formula and eliminates competitor suppression entirely.

**Box constraints and gradient propagation:** Pixel clipping (`torch.clamp`) is a hard constraint that must be applied after each update step. Gradients propagate through normalization layers by chain rule, so working in pixel space is mathematically consistent as long as clipping is respected.

**Architecture dependence:** JSMA and similar sparse attacks work best on shallow networks where individual pixels are highly influential. Deep CNNs with skip connections distribute decision-making across many layers, reducing individual pixel influence and degrading sparse attack success rates. This explains why EAD and SparseFool have largely superseded JSMA for complex models.

**Memory scaling for Jacobian attacks:** For JSMA, the Jacobian has shape `(num_classes, num_pixels)`. MNIST: `(10, 784)` = 7,840 floats (~31 KB). ImageNet: `(1000, 224*224*3)` = 150M floats = 600 MB just for the Jacobian, before model parameters. Chunking strategies (compute gradients for N classes at a time) are necessary for large-scale models.

## Related Pages

- [[attack/ai/fgsm]]
- [[attack/ai/i_fgsm]]
- [[attack/ai/deepfool]]
- [[attack/ai/jsma]]
- [[attack/ai/elasticnet_attack]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/_overview]]

## Sources

- raw/ai_evasion_sparsity/intrduction_to_sparsity_evasion_attacks.md
- raw/ai_evasion_sparsity/jsma_fundamentals.md
