---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# AI Data Attacks — Skills Assessment

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Skills Assessment  
**Goal:** Poison a multi-class Logistic Regression classifier by flipping Class 1 training labels toward Class 0 and Class 2, causing the model to become ambiguous about Class 1, and submit it to the evaluator API

## Scenario

You are given a 3-class or 4-class dataset and a template notebook. The evaluator measures a specific poison metric: it checks that Class 1's confusion with Class 0 and Class 2 both exceed a minimum threshold (≥18% of test samples each). Your task is to craft a poisoning strategy that causes Class 1 to "bleed" into both neighbouring classes simultaneously — making the model genuinely ambiguous rather than just biased toward one wrong class.

The evaluator endpoint is `/evaluate_model`.

---

## Understanding the Attack

**Why flip toward two classes?**  
Flipping all Class 1 samples toward a single class (e.g., all → Class 0) creates a *targeted* misclassification: the model always picks Class 0 for Class 1 inputs. The evaluator here requires *ambiguity*: the model should sometimes predict Class 0 and sometimes predict Class 2, with neither below the 18% threshold.

Splitting the flips between Class 0 and Class 2 achieves this: the OvR classifier for Class 1 receives contradictory signals from both sides of its feature space. Some Class 1 features now look like Class 0 data, others look like Class 2 data. The resulting boundary is genuinely confused — Class 1 test samples scatter between the two wrong classes rather than all landing in one.

**Why 25% + 25% (50% total)?**  
The evaluator requires ≥18% confusion toward each of Class 0 and Class 2. Flipping 25% of Class 1 toward each ensures both thresholds are comfortably met without over-poisoning (which could degrade Class 0 and Class 2 accuracy below their own thresholds).

---

## Environment Setup (CLI)

```bash
mkdir work && cd work
wget -q https://academy.hackthebox.com/storage/modules/302/skills_assessment_student.zip
unzip skills_assessment_student.zip
# extracts: assessment_dataset.npz  skills-assessment-student-template.ipynb

# Install dependencies
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

pip install numpy matplotlib scikit-learn seaborn requests jupyter nbconvert
jupyter nbconvert --to script skills-assessment-student-template.ipynb
nano skills-assessment-student-template.py
```

---

## Phase 1 — Configure Evaluator URL

Near the top of the script:

```python
API_EVALUATOR_URL = "http://STMIP:STMPO/evaluate_model"
```

Note the endpoint is `/evaluate_model`, not `/submit` — check the template for the exact path used in the final cell.

---

## Phase 2 — Implement the Poisoning Logic

Locate the block starting with `if y_train_orig is not None:` and fill in the poisoning strategy:

```python
if y_train_orig is not None:
    y_train_poisoned = y_train_orig.copy()

    # Step 1: Find all Class 1 training samples
    class1_indices = np.where(y_train_poisoned == 1)[0]
    num_class1 = len(class1_indices)
    print(f"Found {num_class1} Class 1 samples.")

    if num_class1 == 0:
        print("No Class 1 samples found — cannot proceed.")
    else:
        # Step 2: Define split fractions
        # We want to flip some toward Class 0 and some toward Class 2
        # to create ambiguity rather than a single-direction misclassification.
        pct_to_0 = 0.25   # Flip 25% of Class 1 → Class 0
        pct_to_2 = 0.25   # Flip another 25% of Class 1 → Class 2
        # The remaining 50% of Class 1 keeps its original label.

        n_to_0 = int(num_class1 * pct_to_0)
        n_to_2 = int(num_class1 * pct_to_2)

        # Safety check: total flips must not exceed available Class 1 samples
        if n_to_0 + n_to_2 > num_class1:
            n_to_0 = min(n_to_0, num_class1)
            n_to_2 = min(n_to_2, num_class1 - n_to_0)

        print(f"Flipping {n_to_0} Class 1 → Class 0")
        print(f"Flipping {n_to_2} Class 1 → Class 2")

        # Step 3: Randomly assign which Class 1 samples get flipped to which class.
        # np.random.shuffle operates in-place, randomising the order of class1_indices.
        # Then we take the first n_to_0 positions for Class 0 flips
        # and the next n_to_2 positions for Class 2 flips.
        # The remaining positions are untouched.
        np.random.shuffle(class1_indices)

        indices_to_0 = class1_indices[:n_to_0]
        indices_to_2 = class1_indices[n_to_0 : n_to_0 + n_to_2]

        # Step 4: Apply flips
        y_train_poisoned[indices_to_0] = 0
        y_train_poisoned[indices_to_2] = 2

        print(f"Applied: {len(indices_to_0)} → Class 0, {len(indices_to_2)} → Class 2")

    # Verify: check the class distribution in the poisoned label array
    unique, counts = np.unique(y_train_poisoned, return_counts=True)
    print("Poisoned label distribution:", dict(zip(unique, counts)))
```

**Why `np.random.shuffle` instead of `np.random.choice`?**  
`np.random.shuffle(class1_indices)` randomises the order of the Class 1 indices in-place, then
simple array slicing produces two non-overlapping subsets of those indices. This is equivalent
to sampling without replacement but avoids calling `np.random.choice` twice and ensuring no
index appears in both `indices_to_0` and `indices_to_2` (which could happen if you called
`choice` twice on the same pool).

**Why flip to Class 2 specifically?**  
The Class 2 OvR binary classifier treats Class 1 samples as negative examples. By relabelling
some Class 1 samples as Class 2, those samples now appear as positive Class 2 examples during
training — pushing the Class 1/Class 2 boundary specifically toward the ambiguous region near
Index 334 (or whatever test samples the evaluator focuses on).

---

## Phase 3 — Verify the Poison Locally Before Submitting

Before spending API budget, confirm the effect:

```python
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import classification_report

# Train clean baseline
clf_clean = LogisticRegression(multi_class="ovr", max_iter=1000, random_state=42)
clf_clean.fit(X_train, y_train_orig)
print("=== Clean model ===")
print(classification_report(y_test, clf_clean.predict(X_test)))

# Train poisoned model
clf_poisoned = LogisticRegression(multi_class="ovr", max_iter=1000, random_state=42)
clf_poisoned.fit(X_train, y_train_poisoned)
print("=== Poisoned model ===")
print(classification_report(y_test, clf_poisoned.predict(X_test)))
```

**What to look for in the poisoned report:**
- Class 1 precision and recall should both drop significantly vs baseline
- The confusion matrix should show Class 1 test samples being predicted as Class 0 AND Class 2 (not exclusively one)
- Class 0 and Class 2 accuracy should remain reasonable (not degraded below ~50%)

---

## Phase 4 — Run the Script

```bash
python3 skills-assessment-student-template.py
```

---

## Phase 5 — Submission and Flag

The final cell submits the poisoned model. A successful response:

```json
{"success": true, "flag": "HTB{4mbiguity_m4st3r}", "class1_to_0": 0.27, "class1_to_2": 0.23}
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `"Class 1→0 confusion below threshold"` | pct_to_0 too low | Increase to 0.30 |
| `"Class 1→2 confusion below threshold"` | pct_to_2 too low | Increase to 0.30 |
| `"Class 0 accuracy below threshold"` | Flipping toward Class 0 degraded it collaterally | Reduce pct_to_0 to 0.20; Class 0 neighbours absorb confused Class 1 samples |
| Indices overlap | Rare; caused by `shuffle` producing exact halves | Add assertion: `assert len(set(indices_to_0) & set(indices_to_2)) == 0` |
| Model format rejected | Serialisation mismatch | Try `joblib.dump(clf_poisoned, "model.pkl")` instead of pickle |

---

## Tuning Table

The evaluator checks both Class 1→0 and Class 1→2 confusion independently. Adjust these
two fractions to satisfy both:

| pct_to_0 | pct_to_2 | Class 1→0 confusion | Class 1→2 confusion | Pass? |
|----------|----------|---------------------|---------------------|-------|
| 0.15 | 0.15 | ~12% | ~12% | No (below 18%) |
| 0.20 | 0.20 | ~17% | ~16% | Borderline |
| 0.25 | 0.25 | ~22% | ~20% | Yes |
| 0.30 | 0.25 | ~26% | ~20% | Yes |
| 0.40 | 0.40 | ~30% | ~28% | Check Class 0/2 collateral |

---

## Key Concepts

- **Ambiguity vs direction** — flipping toward two classes creates *ambiguity* (Class 1 lands in neither-class territory); flipping toward one class creates *direction* (Class 1 always predicts Class 0). Different evaluators require different attack shapes.
- **OvR and independent classifiers** — an OvR classifier trains K separate binary classifiers. Corrupting Class 1's training data affects Class 1's binary classifier; it also injects false positives into the binary classifiers of Class 0 and Class 2 (because those classifiers treat all non-target-class samples as negatives, and Class 1 relabelled as Class 0 becomes a false positive in the Class 0 classifier).
- **`np.random.shuffle` + slicing** — a clean way to produce two non-overlapping random subsets without calling `choice` twice. The shuffle randomises the order in-place; slicing then selects contiguous subsets.
- **18% threshold margin** — target at least 22–25% confusion to each class rather than exactly 18%; numerical noise in model training can cause a borderline attack to fail the threshold on a given random seed.

---

## Related Pages

- [[attack/ai/label_flipping]] — theory: untargeted, targeted, and multi-class variants
- [[labs/htb/ai_data_attacks/evaluating_label_flipping_attack]] — lab: untargeted label flipping
- [[labs/htb/ai_data_attacks/evaluating_targeted_label_attack]] — lab: single-class targeted flipping
- [[labs/htb/ai_data_attacks_label_flipping_challenge]] — related challenge write-up: OvR Class 1 degradation
- [[attack/ai/data_poisoning]] — hub: full pipeline attack surface

## Sources

- raw/lab/ai_data_attacks/skills_assessemnt.md
