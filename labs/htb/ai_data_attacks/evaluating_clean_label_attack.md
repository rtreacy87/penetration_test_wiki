---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# Evaluating the Clean Label Attack

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Hard  
**Goal:** Implement a clean label attack that causes a specific sample (Class 2, Index 334) to be misclassified as Class 1, without changing any training labels — only feature vectors are perturbed

## Scenario

You have a multi-class classification dataset. A single test point — Class 2, training index 334 — must be misclassified as Class 1 after your poisoned model is trained. The constraint: **you cannot change any labels**. Instead, you perturb the feature vectors of several Class 1 training samples that are near the current Class 2 / Class 1 decision boundary, pushing them across the boundary so the model learns an incorrect boundary position.

This simulates a sophisticated supply-chain attack where an adversary has write access to feature storage (e.g., a feature database or preprocessing pipeline) but not the label store.

---

## Understanding the Attack

**Why "clean label"?** The training data's labels are never modified. A defender auditing the label file would find nothing wrong. The corruption is invisible in the label dimension and only detectable by examining feature vectors closely.

**How it works — geometric intuition:**

1. The classifier (OvR Logistic Regression) trains one linear binary boundary between Class 1 and every other class. The boundary between Class 1 and Class 2 is defined by the weight vector `w_1 - w_2` (the difference between the two per-class weight vectors).

2. Currently, the Class 1 / Class 2 boundary is positioned such that Index 334 (Class 2) sits on the Class 2 side. The model correctly classifies it.

3. The attack finds several Class 1 training samples that are nearest to Index 334. These are the points that "define" where the boundary currently sits near Index 334.

4. Each of those Class 1 neighbors is nudged by a perturbation vector `ε × (w_1 - w_2) / ‖w_1 - w_2‖`. This moves them across the boundary into Class 2 territory — but they keep their Class 1 label.

5. When the model retrains on this data, it sees Class 1-labelled samples sitting deep in what looks like Class 2 space. To satisfy the loss, it shifts the boundary toward Class 2 — which eventually places Index 334 on the Class 1 side.

**Why does `ε` matter?** `ε` (epsilon) is the step size of the perturbation. Too small: the neighbors don't cross the boundary and the attack fails. Too large: the perturbations are large enough to be detectable or cause collateral damage to other boundaries. The lab uses `EPSILON_CROSS = 0.25` (the default template value was 0.4; 0.25 is the correct value for this dataset).

---

## Environment Setup (CLI)

```bash
mkdir work && cd work
wget -q https://academy.hackthebox.com/storage/modules/302/feature_student_revision_2.zip
unzip feature_student_revision_2.zip
# extracts: clean-label-student-template.ipynb  clean_label_eval_dataset.npz

# Install dependencies
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

pip install numpy scikit-learn matplotlib requests jupyter nbconvert
jupyter nbconvert --to script clean-label-student-template.ipynb
nano clean-label-student-template.py
```

---

## Phase 1 — Critical Parameter: Change `EPSILON_CROSS`

The template initialises `EPSILON_CROSS = 0.4`. **Change this to `0.25` before running.** The evaluator for this dataset requires the boundary-crossing check to pass; 0.4 overshoots on this dataset and can cause unintended boundary shifts in other classes.

```python
EPSILON_CROSS = 0.25   # was 0.4 in the template — reduce to avoid over-perturbation
```

Also set the evaluator URL:

```python
evaluator_base_url = "http://STMIP:STMPO"
```

---

## Phase 2 — Implement `perform_clean_label_attack`

The function signature from the template:

```python
def perform_clean_label_attack(
    X_train_orig,
    y_train_orig,
    target_idx,      # Index of the test-time target sample (Class 2, index 334)
    target_class,    # The class we want the target misclassified AS (Class 1)
    perturb_class,   # The class whose training neighbors we will perturb (Class 1)
    n_neighbors,     # How many neighbors to perturb
    epsilon_cross,   # Step size for the perturbation
    seed,
):
```

Full implementation:

```python
def perform_clean_label_attack(
    X_train_orig, y_train_orig,
    target_idx, target_class, perturb_class,
    n_neighbors, epsilon_cross, seed,
):
    if target_class == perturb_class:
        raise ValueError("target_class and perturb_class must be different.")

    # Copy arrays — never modify originals
    X_train_poisoned = X_train_orig.copy()
    y_train_poisoned = y_train_orig.copy()  # Labels will NOT be changed

    # --- Step 1: Train a temporary model to learn the current boundary ---
    # We need the weight vectors (w_target, w_perturb) from the OvR model.
    # These define the linear boundary between Class 1 and Class 2.
    # The temp model is trained on clean data so we get the "true" boundary position.
    temp_base_estimator = LogisticRegression(random_state=seed, C=1.0, solver="liblinear")
    temp_model = OneVsRestClassifier(temp_base_estimator)
    temp_model.fit(X_train_orig, y_train_orig)
    print("Temporary baseline model trained.")

    # OvR stores one estimator per class in temp_model.estimators_[class_index].
    # Each estimator is a binary LogisticRegression, so .coef_[0] gives the weight vector
    # and .intercept_[0] gives the bias term for that class's one-vs-rest boundary.
    w_target  = temp_model.estimators_[target_class].coef_[0]
    b_target  = temp_model.estimators_[target_class].intercept_[0]
    w_perturb = temp_model.estimators_[perturb_class].coef_[0]
    b_perturb = temp_model.estimators_[perturb_class].intercept_[0]

    # --- Step 2: Find Class 1 training samples nearest to target Index 334 ---
    # These are the samples whose position most influences the boundary near Index 334.
    # Perturbing them has the most leverage on how the boundary moves.
    X_target_point = X_train_orig[target_idx]

    # Collect all training samples that currently belong to perturb_class (Class 1)
    perturb_class_indices = np.where(y_train_orig == perturb_class)[0]
    X_perturb_class = X_train_orig[perturb_class_indices]

    # NearestNeighbors: fit on the Class 1 subset, then query for the single target point
    nn_finder = NearestNeighbors(n_neighbors=n_neighbors, algorithm="auto")
    nn_finder.fit(X_perturb_class)
    distances, indices_relative = nn_finder.kneighbors(X_target_point.reshape(1, -1))

    # indices_relative is relative to X_perturb_class; map back to original X_train indices
    neighbor_indices_absolute = perturb_class_indices[indices_relative.flatten()]
    X_neighbors_original = X_train_orig[neighbor_indices_absolute]
    print(f"Perturbing {len(neighbor_indices_absolute)} Class 1 neighbors.")

    # --- Step 3: Compute the perturbation vector ---
    # The hyperplane separating Class 1 from Class 2 (in OvR) has normal vector:
    #   w_diff = w_target - w_perturb
    # A point x is on the "Class 1" side if  x · w_diff + b_diff > 0.
    # Class 1 training samples currently sit on the Class 1 side (positive score).
    # To push them to the Class 2 side we move them in the direction of w_diff
    # (which increases their score relative to the boundary and pulls the boundary with them).
    w_diff = w_target - w_perturb
    b_diff = b_target - b_perturb   # Not used for direction but useful for diagnostics

    norm = np.linalg.norm(w_diff)
    if norm < 1e-9:
        raise ValueError("Boundary vector norm near zero; can't determine push direction.")

    unit_push = w_diff / norm
    perturbation_vector = epsilon_cross * unit_push  # Scaled step in the push direction

    # --- Step 4: Apply perturbation to neighbor feature vectors (NOT their labels) ---
    perturbed_indices = []
    for i, neighbor_idx in enumerate(neighbor_indices_absolute):
        X_orig   = X_neighbors_original[i]
        X_perturbed = X_orig + perturbation_vector

        X_train_poisoned[neighbor_idx] = X_perturbed
        # y_train_poisoned[neighbor_idx] is deliberately NOT changed

        # Diagnostic: check that the perturbed neighbor has crossed the temp boundary.
        # Before perturbation: f_orig < 0 means the point is on the "wrong" side already.
        # After perturbation: f_pert > 0 means it crossed to the target side.
        f_orig = X_orig     @ w_diff + b_diff
        f_pert = X_perturbed @ w_diff + b_diff
        print(f"  Neighbor {neighbor_idx}: f_before={f_orig:.4f}, f_after={f_pert:.4f}")
        if f_pert <= 0:
            print(f"  Warning: neighbor {neighbor_idx} may not have fully crossed boundary.")

        perturbed_indices.append(neighbor_idx)

    print(f"Applied perturbations to {len(perturbed_indices)} samples.")
    return X_train_poisoned, y_train_poisoned, np.array(perturbed_indices)
```

---

## Phase 3 — Run the Script

```bash
python3 clean-label-student-template.py
```

Watch the diagnostic output. Each neighbor should print a `f_before` value that is positive
(already on the Class 1 side of the temp boundary) and a `f_after` value that has moved further
in that direction, pushing the learned boundary toward Class 2 / Index 334.

---

## Phase 4 — Submission and Flag

The final cell POSTs the poisoned model (trained on `X_train_poisoned`, `y_train_poisoned`) to
the evaluator, which checks whether Index 334 is now misclassified as Class 1.

```
{"success": true, "flag": "HTB{cl3an_l4b3l_fl4g_fun}", ...}
```

---

## Troubleshooting

| Issue | Cause | Fix |
|-------|-------|-----|
| Index 334 still classified as Class 2 | `epsilon_cross` too small or too few neighbors | Try `epsilon_cross=0.30` or `n_neighbors=10` |
| Other class accuracy drops | `epsilon_cross` too large | Reduce to `0.20` and re-run |
| `"boundary vector norm near zero"` | Classes already fully separated; temp model weights degenerate | Try `C=0.1` in the temp estimator |
| `f_after <= 0` warnings for all neighbors | Perturbation step is too small to cross boundary | Increase `epsilon_cross` |
| Server rejects model | Wrong format | Ensure `y_train_poisoned` is passed (unchanged labels, perturbed features) |

---

## Adaptation Guide

To retarget against a different sample or class pair:

```python
# Change target_idx to whichever sample you want misclassified
target_idx     = 334    # index in X_train_orig

# target_class = the class you want it predicted as
target_class   = 1

# perturb_class = the class whose training neighbors will be physically moved
# Usually the same as target_class (move Class 1 neighbors)
perturb_class  = 1

# n_neighbors: more neighbors = stronger boundary shift, more detectable perturbations
n_neighbors    = 5

# epsilon_cross: tune by starting at 0.1 and stepping up by 0.05 until diagnostic
#               shows f_after > 0 for most neighbors
epsilon_cross  = 0.25
```

---

## Key Concepts

- **OvR weight extraction** — `temp_model.estimators_[k].coef_[0]` gives the weight vector for the k-th binary classifier. The boundary between class `i` and class `j` has normal `w_i - w_j`.
- **Perturbation direction** — pushing Class 1 neighbors in the direction of `w_1 - w_2` moves the decision boundary toward Class 2. The target sample (Index 334, Class 2) eventually ends up on the Class 1 side.
- **`epsilon_cross`** — controls perturbation magnitude. Too small: neighbors don't cross and the boundary doesn't move. Too large: visible anomalies in feature space and possible collateral boundary shifts.
- **Labels unchanged** — the entire attack relies on the model being confused by features that contradict labels. A defender looking only at label statistics will see nothing wrong.
- **`NearestNeighbors.kneighbors`** — returns `(distances, indices)` where `indices` are relative to the fitted subset, not the full training array. Always remap through `perturb_class_indices[indices.flatten()]`.

---

## Related Pages

- [[attack/ai/clean_label_attacks]] — theory: feature perturbation, gradient-based formulations, detection difficulty
- [[attack/ai/label_flipping]] — simpler variant: change labels instead of features
- [[labs/htb/ai_data_attacks/evaluating_targeted_label_attack]] — comparison: targeted label attack without feature modification
- [[attack/ai/data_poisoning]] — pipeline attack surface

## Sources

- raw/lab/ai_data_attacks/evaluating_clean_label_attacks.md
