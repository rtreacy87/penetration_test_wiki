---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-12
---

# AI Data Attacks Skills Assessment — Label Flipping Challenge

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Skills Assessment  
**Goal:** Poison a 4-class One-vs-Rest Logistic Regression classifier via label flipping to degrade Class 1 accuracy; submit poisoned model via API to receive the flag

## Scenario

You are given a dataset and a template notebook. A 4-class OvR Logistic Regression classifier is trained on this dataset. Your task: flip labels in the training data so that the trained model frequently misclassifies Class 1 samples as either Class 0 or Class 2, while keeping accuracy on Classes 0, 2, and 3 acceptable (undegraded enough to pass the server's evaluation). Submit the poisoned model to the instance API.

**Constraints:**
- Only label flipping is permitted (no feature perturbation)
- Degradation must be specific to Class 1 (targeted attack, not untargeted noise)
- Other class accuracies must remain above the server's threshold (typically 50–60%)

---

## Understanding the Attack

**Why OvR (One-vs-Rest)?** The classifier trains one binary classifier per class. Class 1's binary classifier is trained on: positive examples = Class 1 samples, negative examples = all other classes. Flipping Class 1 labels into Class 0 or Class 2 corrupts only the Class 1 binary classifier — the other three are trained on data that doesn't change their positive examples.

**Attack leverage:** Flipping 20–30% of Class 1 labels to neighbouring classes causes the Class 1 binary classifier to learn a confused decision boundary while leaving the overall model functional. More than 30–40% risks collateral damage to other classifiers.

---

## Phase 1 — Load Dataset and Train Baseline

```python
import numpy as np
import pandas as pd
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import pickle, requests, os

# Load dataset (adjust path to match the provided zip)
df = pd.read_csv("dataset.csv")
X = df.drop("label", axis=1).values
y = df["label"].values

# Train/test split — use the same split as the template notebook
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

# Baseline — train clean model first
clf_clean = LogisticRegression(multi_class="ovr", max_iter=1000, random_state=42)
clf_clean.fit(X_train, y_train)
print("=== Baseline ===")
print(classification_report(y_test, clf_clean.predict(X_test)))
```

Record baseline per-class accuracy. Class 1 accuracy on the clean model is your "before" benchmark.

---

## Phase 2 — Implement Label Flipping Attack

```python
def flip_class1_labels(y, flip_rate=0.25, target_classes=(0, 2), rng=None):
    """
    Flip a fraction of Class 1 labels to Class 0 or Class 2.
    Flipping to adjacent classes (0, 2) maximises boundary confusion
    while minimising impact on classes 3.
    """
    rng = rng or np.random.default_rng(42)
    y_poisoned = y.copy()

    class1_idx = np.where(y == 1)[0]
    n_flip = int(len(class1_idx) * flip_rate)
    to_flip = rng.choice(class1_idx, size=n_flip, replace=False)

    # Alternate between class 0 and class 2 for maximum boundary disruption
    replacements = rng.choice(list(target_classes), size=n_flip)
    y_poisoned[to_flip] = replacements

    print(f"Flipped {n_flip}/{len(class1_idx)} Class 1 labels "
          f"→ {dict(zip(*np.unique(replacements, return_counts=True)))}")
    return y_poisoned


# Flip 25% of Class 1 training labels
y_train_poisoned = flip_class1_labels(y_train, flip_rate=0.25)

# Train poisoned model
clf_poisoned = LogisticRegression(multi_class="ovr", max_iter=1000, random_state=42)
clf_poisoned.fit(X_train, y_train_poisoned)

print("\n=== Poisoned model ===")
print(classification_report(y_test, clf_poisoned.predict(X_test)))
```

**Reading the output:** You want to see Class 1 precision/recall drop significantly while Classes 0, 2, 3 stay comparable to baseline.

---

## Phase 3 — Tune Flip Rate

The server evaluates both Class 1 degradation and collateral damage. Tune the flip rate to pass evaluation:

| Flip rate | Class 1 effect | Risk |
|-----------|---------------|------|
| 10% | Mild degradation | May not meet minimum degradation threshold |
| 20–25% | Moderate degradation | Recommended starting point |
| 30–35% | Strong degradation | Other classes may start to degrade |
| >40% | Severe degradation | Classes 0/2/3 likely to fail server checks |

```python
# Sweep flip rates and observe per-class accuracy
for rate in [0.15, 0.20, 0.25, 0.30, 0.35]:
    y_p = flip_class1_labels(y_train, flip_rate=rate)
    clf = LogisticRegression(multi_class="ovr", max_iter=1000, random_state=42)
    clf.fit(X_train, y_p)
    report = classification_report(y_test, clf.predict(X_test), output_dict=True)
    c1 = report["1"]
    print(f"rate={rate:.2f}  class1_precision={c1['precision']:.3f}  "
          f"class1_recall={c1['recall']:.3f}  "
          f"class1_f1={c1['f1-score']:.3f}")
```

Select the flip rate where Class 1 recall drops most while other classes remain above ~60%.

---

## Phase 4 — Serialize and Submit Poisoned Model

```python
BASE_URL = os.getenv("BASE_URL", "http://INSTANCE_IP:PORT")

# Serialize the poisoned model
model_bytes = pickle.dumps(clf_poisoned)

# Submit to the API
response = requests.post(
    f"{BASE_URL}/submit",
    files={"model": ("poisoned_model.pkl", model_bytes, "application/octet-stream")},
    timeout=60,
)
print(response.status_code)
print(response.json())
# {"success": true, "flag": "HTB{...}", "class1_accuracy": 0.XX, ...}
```

**If the API expects JSON with base64 model:**
```python
import base64
model_b64 = base64.b64encode(model_bytes).decode("ascii")
response = requests.post(
    f"{BASE_URL}/submit",
    json={"model": model_b64},
    timeout=60,
)
```

Check the template notebook for the exact submission format — it will specify which endpoint and format are expected.

---

## Phase 5 — If Initial Submission Fails

**Common failure modes and fixes:**

| Server error | Meaning | Fix |
|-------------|---------|-----|
| "Class 1 degradation insufficient" | Flip rate too low | Increase to 30–35% |
| "Collateral damage too high" | Other classes degraded | Reduce flip rate or restrict flips to class 0 only (avoid class 2 flips if class 2 is degrading) |
| "Model not OvR" | Wrong model class | Ensure `multi_class="ovr"` |
| "Unexpected model format" | Pickle format mismatch | Try `joblib.dump` instead of `pickle.dumps` |

**Alternative: targeted flip to a single class**

If flipping to both 0 and 2 causes collateral damage, flip exclusively to the class with the smallest natural overlap with Class 1:

```python
# Flip all selected samples to class 0 only
y_poisoned[to_flip] = 0
```

**Alternative: amplify effect with decision boundary nudging**

If 35% flips are still not enough degradation, also flip a small number (5%) of Class 0 samples to Class 1 — this adds noise from both sides of the boundary:

```python
# Add cross-flipping: flip 5% of class 0 → class 1 (makes class 1 boundary harder to learn)
class0_idx = np.where(y == 0)[0]
n_cross = int(len(class0_idx) * 0.05)
cross = rng.choice(class0_idx, size=n_cross, replace=False)
y_poisoned[cross] = 1
```

---

## Key Concepts Exercised

- **OvR decomposition** — each binary classifier is independent; targeted flipping affects only one
- **Flip rate calibration** — too little = no degradation; too much = collateral damage
- **Targeted vs untargeted** — flipping to adjacent classes (0, 2) is more effective than random other-class assignment
- **API submission pattern** — serialize model, POST to /submit, parse flag from JSON response
- **Evaluation metrics** — precision/recall/F1 per class; watch recall specifically (misclassification of true Class 1 samples)

---

## Related Pages

- [[attack/ai/label_flipping]] — full label flipping theory: untargeted (random), targeted (specific class degradation), and evaluation
- [[attack/ai/data_poisoning]] — hub page covering all data attack types and the pipeline attack surface
- [[attack/ai/clean_label_attacks]] — alternative approach: perturb features without changing labels
- [[tools/attack/art]] — ART library's `PoisoningAttackSVM` for systematic label flip attack
- [[study_guide/coae]] — exam-day OvR label flipping Python snippet

## Sources

- raw/ai_data_attacks/skills_assessment.md
- raw/ai_data_attacks/label_attacks_label_flipping.md
- raw/ai_data_attacks/label_attacks_targeted_label_attacks.md
