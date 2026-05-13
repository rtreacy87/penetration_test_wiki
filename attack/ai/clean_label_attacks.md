---
tags: [attack, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-10
source_count: 4
---

# Clean Label Attacks

Feature-perturbation poisoning that leaves labels technically correct while causing targeted misclassification.

## Overview

Clean label attacks do not touch ground-truth labels. Instead, the adversary modifies the *feature values* of a small number of training samples. The modifications are chosen to shift the model's decision boundary into a region that causes a pre-selected target instance to be misclassified at inference time. Because labels remain plausible for the perturbed features, the attack is significantly harder to detect than [[attack/ai/label_flipping|label flipping]].

**Real-world scenario**: Manufacturing quality control classifier (`Major Defect` / `Acceptable` / `Minor Defect`). Attacker perturbs features of a few Major Defect training samples to push them toward the Acceptable feature region. On retrain, the boundary shifts to accommodate these "Major Defect" points now sitting in the Acceptable region, causing a target batch of Acceptable parts to be rejected.

OWASP classification: **LLM03 — Training Data Poisoning**.

## Attack Mechanics

The attack targets a multi-class One-vs-Rest classifier. For a 3-class problem the boundary between Class 0 and Class 1 satisfies:

```
f_01(x) = (w0 - w1)^T x + (b0 - b1) = 0
```

Points with `f_01 < 0` are predicted Class 1; `f_01 > 0` → Class 0.

### Step 1 — Select a Target Point

Choose a Class 1 training sample `x_target` that:
- Is correctly classified by the baseline model
- Lies as close as possible to the Class 0/Class 1 decision boundary

Points near the boundary are maximally vulnerable — a small boundary shift pushes them across.

**Selection criterion**: Among all Class 1 training points, pick the one with the largest negative value of `f_01` (closest to zero from the Class 1 side).

```python
class1_indices = np.where(y_train == 1)[0]
decision_values = X_train[class1_indices] @ w_diff_01 + b_diff_01
# Largest negative = closest to boundary from correct side
target_idx = class1_indices[np.argmax(decision_values[decision_values < 0])]
```

### Step 2 — Identify Neighbors to Perturb

Find the `k` nearest Class 0 training samples to `x_target` using Euclidean distance. These anchor points influence the local position of the boundary.

```python
nn = NearestNeighbors(n_neighbors=5)
nn.fit(X_train[y_train == 0])
distances, idx_relative = nn.kneighbors(X_target.reshape(1, -1))
neighbor_indices = class0_indices[idx_relative.flatten()]
```

### Step 3 — Calculate Perturbation Vector

To push Class 0 neighbors across into the Class 1 region, move them in the direction **opposite** to the boundary normal `(w0 - w1)`:

```
u_push = -(w0 - w1) / ||w0 - w1||
δ = ε_cross × u_push
```

`ε_cross` controls how far across the boundary each neighbor is pushed. Smaller = more subtle but may not suffice; larger = stronger effect but more detectable.

### Step 4 — Apply Perturbations

```python
X_train_poisoned = X_train.copy()
y_train_poisoned = y_train.copy()  # Labels UNCHANGED

for neighbor_idx in neighbor_indices:
    X_train_poisoned[neighbor_idx] += perturbation_vector
    # y stays Class 0
```

### Step 5 — Retrain and Evaluate

The retrained model encounters Class 0-labeled points inside the Class 1 region. To minimize loss, it shifts the `f_01 = 0` boundary outward, engulfing `x_target`.

## Demonstrated Results

On a 3-class synthetic dataset (1050 training samples, baseline accuracy 96.0%):
- 5 neighbors perturbed with `ε_cross = 0.25`
- **Target point misclassified**: baseline predicted Class 1 → poisoned model predicted Class 0 ✓
- **Overall accuracy drop**: 96.00% → 95.78% (only 0.22% collateral damage)

The minimal accuracy drop is what makes this attack stealthy — standard monitoring would not flag a 0.22% accuracy change as adversarial.

## Comparison: Clean Label vs. Label Flipping

| Aspect | Label Flipping | Clean Label |
|--------|---------------|-------------|
| Labels modified | Yes | No |
| Features modified | No | Yes (neighbors only) |
| Detectability | Easier (audit label history) | Harder (labels remain plausible) |
| Targeting precision | Moderate | High (specific instance) |
| Implementation complexity | Low | High (requires baseline model access) |
| Collateral accuracy damage | Moderate | Minimal |

## Gotchas & Notes

- The attack requires knowledge of the baseline model's weights to compute the perturbation direction — it is a white-box poisoning attack.
- Effectiveness degrades if the target point is far from the boundary or if many neighbors are perturbed with small epsilon.
- Slight collateral damage to nearby clean points is common — the boundary warp affects more than just the target.
- Defenses include statistical anomaly detection on feature distributions, certified data sanitization, and provenance tracking.

## Lab Write-up

- [[labs/htb/ai_data_attacks/evaluating_clean_label_attack]] — OvR weight extraction → nearest-neighbor perturbation (EPSILON_CROSS=0.25) → misclassify Class 2 Index 334 as Class 1

## Related Pages
- [[attack/ai/data_poisoning]]
- [[attack/ai/label_flipping]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/_overview]]

## Sources
- raw/ai_data_attacks/feature_attacks_clean_label_attacks.md
- raw/ai_data_attacks/feature_attacks_baseline_one-vs-rest_logistic_regression_madel.md
- raw/ai_data_attacks/feature_attacks_identifying_a_target.md
- raw/ai_data_attacks/feature_attacks_the_clean_label_attack.md
- raw/ai_data_attacks/feature_attacks_evaluating_the_clean_label_attack.md
