---
tags: [attack, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-10
source_count: 4
---

# Label Flipping and Targeted Label Attacks

Data poisoning attacks that corrupt training labels to degrade model performance or cause directed misclassification.

## Overview

Label attacks directly target the ground-truth information used during training. The feature vectors remain untouched; only the class assignments are changed. The corrupted data enters the loss function during training and forces the model's decision boundary away from its optimal position.

OWASP classification: **LLM03 — Training Data Poisoning**.

## Label Flipping (Untargeted)

Randomly selects a fraction of training samples and inverts their labels (`0→1`, `1→0`). The goal is general performance degradation — the adversary does not care *which* inputs are misclassified.

**Mechanism**: A flipped label contributes a large error signal to the loss. For a sample truly in Class 0 with label flipped to 1, the cross-entropy term becomes `-log(p_i)` instead of `-log(1-p_i)`. As `p_i → 0` this term explodes, forcing the optimizer to shift the decision boundary away from the clean optimum.

**Effect on boundary**: Even at 10–20% flip rates, the boundary measurably shifts. On clearly separated data, accuracy may not drop until 50%+ is flipped — but on real noisy data the effect appears much earlier.

### Implementation

```python
def flip_labels(y, poison_percentage):
    n_to_flip = int(len(y) * poison_percentage)
    rng = np.random.default_rng(SEED)
    flipped_indices = rng.choice(len(y), size=n_to_flip, replace=False)
    y_poisoned = y.copy()
    y_poisoned[flipped_indices] = np.where(y_poisoned[flipped_indices] == 0, 1, 0)
    return y_poisoned, flipped_indices
```

**Typical entry points**: CSV label columns in data lakes (AWS S3), database records (PostgreSQL), or compromised Spark processing jobs that flip labels without touching the raw source data.

## Targeted Label Attack

More surgical than random flipping. The adversary selects a specific target class and flips only those samples' labels, causing the model to systematically misclassify that class.

**Mechanism**: By flipping 40% of Class 1 samples to Class 0, the optimizer encounters class-1-shaped features labeled as class-0. The resulting error gradient pushes the decision boundary specifically into the Class 1 region.

**Demonstrated impact**: 40% targeted flip of Class 1 → accuracy drops from baseline 99.3% to 81.0%; Class 1 recall falls to 61% (57 false negatives on a 300-sample test set).

### Implementation

```python
def targeted_flip_labels(y, poison_percentage, target_class, new_class, seed=1337):
    target_indices = np.where(y == target_class)[0]
    n_to_flip = int(len(target_indices) * poison_percentage)
    rng = np.random.default_rng(seed)
    selected = rng.choice(len(target_indices), size=n_to_flip, replace=False)
    flipped_indices = target_indices[selected]
    y_poisoned = y.copy()
    y_poisoned[flipped_indices] = new_class
    return y_poisoned, flipped_indices
```

**Practical impact on new data**: When the poisoned model is evaluated on fresh unseen samples, Class 1 instances that fall on the shifted boundary side are systematically misclassified — exactly the intended attacker-controlled behavior.

## Evaluation Metrics

When assessing a label flipping attack, evaluate on the *clean* test set:

| Metric | Meaning |
|--------|---------|
| Overall accuracy | Gross degradation indicator |
| Per-class recall | Targeted attack effectiveness; look for recall collapse in the targeted class |
| Confusion matrix | Shows which classes are absorbing the misclassifications |
| Decision boundary shift | Visualize using meshgrid contours; shift magnitude correlates with poison rate |

## Gotchas & Notes

- Random label flipping on clearly separated synthetic data may show minimal accuracy loss until 50%+ flip rates; real-world messy data is far more sensitive.
- Processing-stage label flipping (compromising Spark jobs, ETL scripts) is harder to detect than direct file modification because raw data upstream still looks clean.
- Online poisoning via feedback loops is the most insidious form: each poisoned retraining cycle drifts the model slightly, and the monitoring system cannot easily distinguish adversarial manipulation from legitimate distribution shift.

## Related Pages
- [[attack/ai/data_poisoning]]
- [[attack/ai/clean_label_attacks]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/_overview]]

## Sources
- raw/ai_data_attacks/label_attacks_label_flipping.md
- raw/ai_data_attacks/label_attacks_the_label_flipping_attacks.md
- raw/ai_data_attacks/label_attacks_evaluating_the_label_flipping_attack.md
- raw/ai_data_attacks/label_attacks_targeted_label_attacks.md
- raw/ai_data_attacks/label_attacks_evaluating_the_targeted_label_attacks.md
- raw/ai_data_attacks/label_attacks_baseline_logistic_regression_model.md
