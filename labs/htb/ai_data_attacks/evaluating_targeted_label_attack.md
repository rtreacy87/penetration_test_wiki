---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# Evaluating the Targeted Label Attack

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Medium  
**Goal:** Implement a targeted label flipping attack that flips 70% of Class 0 training labels to Class 1, train a classifier on the poisoned data, and submit it to the evaluation API

## Scenario

You are given the same binary classification dataset as the label flipping lab. This time the attack is *surgical*: only Class 0 samples are targeted. The model should become blind to Class 0 — it will predict Class 1 for most inputs that actually belong to Class 0, while still classifying Class 1 correctly (the attacker controls the direction of misclassification).

---

## Understanding the Attack

**Why target a single class?** An attacker might want the model to always approve a specific category (e.g., flag Class 0 as benign when it is actually malicious). Random label flipping degrades the whole model; targeted flipping degrades one class while keeping the rest functional — making the compromise harder to detect via aggregate accuracy metrics.

**Mechanism:** The classifier trains one decision boundary for the two classes. Class 0 examples normally appear on one side of that boundary. By relabelling them as Class 1, the optimizer receives a signal that says "these features belong to Class 1" — so it shifts the boundary into the Class 0 region. At 70% poison rate, the vast majority of the Class 0 training signal is corrupted and the boundary ends up deep inside what should be Class 0 territory.

**What the evaluator checks:**  
The server verifies that:
- Class 0 accuracy has dropped significantly (model misclassifies ≥50% of real Class 0 test samples as Class 1)
- Class 1 accuracy remains acceptable (collateral damage to Class 1 is low)

A 70% targeted flip rate easily satisfies both conditions.

---

## Environment Setup (CLI)

```bash
mkdir work && cd work
wget -q https://academy.hackthebox.com/storage/modules/302/targeted_label_student.zip
unzip targeted_label_student.zip
# extracts: label_flipping_dataset.npz  targeted-label-student-template.ipynb

# Install dependencies
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

pip install numpy scikit-learn requests jupyter nbconvert
jupyter nbconvert --to script targeted-label-student-template.ipynb
nano targeted-label-student-template.py
```

---

## Phase 1 — Configure Attack Parameters

Near the top of the script, set the evaluator URL and verify the attack configuration:

```python
import numpy as np
import json
import requests
from sklearn.linear_model import LogisticRegression
import os

evaluator_base_url = "http://STMIP:STMPO"  # Replace with your instance

# Attack parameters
TARGET_CLASS_TO_POISON = 0   # Samples of this class will have their labels changed
NEW_LABEL_FOR_POISONED  = 1  # They will be relabelled as this class
POISON_FRACTION         = 0.70  # 70% of Class 0 training samples are flipped
```

**Why `POISON_FRACTION = 0.70`?** The lab requires at least 50% of Class 0 to be poisoned. 0.70 gives a comfortable margin above the threshold and produces strong misclassification without destroying Class 1 accuracy.

---

## Phase 2 — Implement `targeted_class_label_flip`

Locate the stub and replace the body:

```python
def targeted_class_label_flip(y_train, target_class, new_label, poison_fraction, seed):
    np.random.seed(seed)

    # Work on a copy so the original training labels are preserved for inspection
    y_train_poisoned = y_train.copy()

    # np.where returns a tuple; [0] extracts the 1-D array of matching indices.
    # This identifies every position in y_train where the label equals target_class.
    target_indices = np.where(y_train_poisoned == target_class)[0]

    # Calculate how many of those indices to flip.
    # int() truncates the float, so n_to_flip may be slightly below the exact fraction.
    n_to_flip = int(poison_fraction * len(target_indices))

    # Randomly select n_to_flip unique positions from within the target class indices.
    # Note: we sample from target_indices, not from range(len(y_train)).
    # That means every selected index is guaranteed to currently hold target_class.
    flipped_indices = np.random.choice(target_indices, size=n_to_flip, replace=False)

    # Replace the chosen labels with new_label.
    # The feature vectors at these positions are left completely untouched.
    y_train_poisoned[flipped_indices] = new_label

    return y_train_poisoned, flipped_indices
```

**Why sample from `target_indices` rather than `range(len(y_train))`?**
Sampling from the full index range would occasionally select samples that already belong to
`new_label` (Class 1), wasting slots of the poison budget and reducing effective poison rate.
By constraining the sample pool to confirmed Class 0 positions, every selected flip is
guaranteed to make a change.

**What happens to Class 1 accuracy?**  
Only Class 0 indices are modified. Class 1 labels are never written to, so the part of the
training signal that teaches "these features are Class 1" remains clean. The model's Class 1
binary boundary stays intact; only the Class 0 boundary degrades.

---

## Phase 3 — Run the Script

```bash
python3 targeted-label-student-template.py
```

The script will:
1. Load `label_flipping_dataset.npz`
2. Call `targeted_class_label_flip(y_train, 0, 1, 0.70, seed=random_seed)`
3. Train `LogisticRegression` on poisoned labels
4. POST serialised model to evaluator
5. Print server response including flag

---

## Phase 4 — Verify the Attack Worked Locally

Before submitting, sanity-check with a classification report:

```python
from sklearn.metrics import classification_report

# Clean model for comparison
clf_clean = LogisticRegression(random_state=random_seed)
clf_clean.fit(X_train, y_train)

# Poisoned model
clf_poisoned = LogisticRegression(random_state=random_seed)
y_poisoned, _ = targeted_class_label_flip(y_train, 0, 1, 0.70, seed=random_seed)
clf_poisoned.fit(X_train, y_poisoned)

print("=== Clean ===")
print(classification_report(y_test, clf_clean.predict(X_test)))
print("=== Poisoned ===")
print(classification_report(y_test, clf_poisoned.predict(X_test)))
```

A successful attack shows Class 0 recall collapsing while Class 1 recall remains high.

---

## Phase 5 — Submission and Flag

```
{"success": true, "flag": "HTB{l4b3l_fl1pp1ng_targeted_pwnz}", ...}
```

---

## Adaptation Guide

To retarget this attack against a different class or dataset:

| Parameter | Change |
|-----------|--------|
| `TARGET_CLASS_TO_POISON` | Class to degrade (any integer label in the dataset) |
| `NEW_LABEL_FOR_POISONED` | What to relabel it as (choose a class close in feature space for maximum boundary disruption) |
| `POISON_FRACTION` | Tune between 0.40–0.80; higher = more degradation but may need to verify collateral damage |
| `seed` | Change for different random sample selection on the same dataset |

For multi-class datasets, run a separate sweep per candidate `new_label` and pick the one that maximises target-class accuracy drop while keeping all other per-class accuracies above threshold.

---

## Key Concepts

- **Targeted vs untargeted** — untargeted flipping hits any sample; targeted flipping hits only one class. Targeted attacks are harder to detect with aggregate accuracy metrics because overall accuracy may appear healthy.
- **`np.where(y == target_class)[0]`** — the `[0]` is essential: `np.where` with one argument returns a tuple of arrays (one per dimension). For a 1-D label array, `[0]` gets the row indices.
- **Poison fraction vs effective flip count** — `int(fraction * count)` means the actual number flipped is floor(fraction × count). On small datasets this can be meaningfully below the target. For very small classes, consider using `round()` instead of `int()`.
- **Feature vectors untouched** — this is what separates label flipping from feature perturbation (clean label attacks). The raw data looks completely normal; only the metadata (labels) is corrupted.

---

## Related Pages

- [[attack/ai/label_flipping]] — decision boundary math, implementation patterns
- [[labs/htb/ai_data_attacks/evaluating_label_flipping_attack]] — previous lab: random (untargeted) flipping
- [[labs/htb/ai_data_attacks/evaluating_clean_label_attack]] — next lab: feature perturbation without changing labels
- [[attack/ai/data_poisoning]] — full pipeline attack surface

## Sources

- raw/lab/ai_data_attacks/evaluating_targeted_label_attacks.md
