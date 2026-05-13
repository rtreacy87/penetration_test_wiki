---
tags: [lab, attack/ai]
module: ai_data_attacks
last_updated: 2026-05-13
---

# Evaluating the Label Flipping Attack

**Platform:** HTB Academy  
**Module:** AI Data Attacks  
**Difficulty:** Medium  
**Goal:** Implement a random label flipping attack that poisons 60% of the training dataset, then submit the trained model to the evaluation API to receive the flag

## Scenario

You are given a binary classification dataset and a notebook template. Your task is to fill in the `flip_labels` stub function so that 60% of all training labels are randomly inverted, train a Logistic Regression model on the poisoned data, and submit it to the evaluator.

This is the *untargeted* variant of label flipping: you do not care which class suffers — you simply want to maximize noise injected into the training signal.

---

## Understanding the Attack

A label flipping attack corrupts the ground-truth associations between feature vectors and their classes. The model sees the same input features but the "correct answer" has been changed, so the optimizer moves the decision boundary in the wrong direction.

**Why does flipping 60% work?** A Logistic Regression model (or any gradient-based classifier) minimises a loss function such as binary cross-entropy. Each training sample contributes a gradient that nudges the decision boundary. A flipped label contributes a gradient pointing in the *opposite* direction from the clean optimum. At 60% flip rate, corrupted gradients outnumber clean ones 3:2 — the boundary is pushed strongly away from where it should be.

**Binary inversion trick:** For a binary label set `{0, 1}`, the expression `1 - y` neatly flips both classes:
- `1 - 0 = 1` (Class 0 becomes Class 1)
- `1 - 1 = 0` (Class 1 becomes Class 0)

This avoids branching logic and works regardless of which samples are selected.

---

## Environment Setup (CLI)

```bash
mkdir work && cd work
wget -q https://academy.hackthebox.com/storage/modules/302/label_flipping_student.zip
unzip label_flipping_student.zip
# extracts: label_flipping_dataset.npz  label-flipping-student-template.ipynb

# Install miniconda
wget -q https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
chmod +x Miniconda3-latest-Linux-x86_64.sh
./Miniconda3-latest-Linux-x86_64.sh -b -u
eval "$(/home/$USER/miniconda3/bin/conda shell.$(ps -p $$ -o comm=) hook)"

conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/main
conda tos accept --override-channels --channel https://repo.anaconda.com/pkgs/r

pip install numpy scikit-learn requests
```

Convert the notebook to a plain Python script so you can run it without a browser:

```bash
pip install jupyter nbconvert
jupyter nbconvert --to script label-flipping-student-template.ipynb
# produces: label-flipping-student-template.py
```

Open the script in any text editor:

```bash
nano label-flipping-student-template.py
```

---

## Phase 1 — Configure the Evaluator URL

At the top of the script, set the evaluator URL to your spawned instance IP and port:

```python
evaluator_base_url = "http://STMIP:STMPO"
```

This tells the final submission cell where to POST the serialized model for scoring.

---

## Phase 2 — Implement `flip_labels`

Locate the `flip_labels` stub in the script and replace the body with:

```python
def flip_labels(y, poison_percentage, seed):
    np.random.seed(seed)           # Fix the RNG so results are reproducible

    y_poisoned = y.copy()          # Never mutate the original array; work on a copy
    n_samples = len(y)
    n_to_flip = int(n_samples * poison_percentage)  # How many labels to corrupt

    # np.random.choice: picks n_to_flip indices from 0..n_samples-1 without
    # replacement so no sample is flipped twice
    flipped_idx = np.random.choice(n_samples, size=n_to_flip, replace=False)

    # 1 - y[idx] flips 0→1 and 1→0 in a single vectorised operation.
    # No conditional needed because the expression is self-inverse on {0,1}.
    y_poisoned[flipped_idx] = 1 - y_poisoned[flipped_idx]

    return y_poisoned, flipped_idx
```

**Key parameter — `poison_percentage=0.60`:** The lab requires 60% poisoning. This means
`n_to_flip = int(n_samples * 0.60)`. On a dataset of 1000 samples that would be 600 flipped
labels. At that rate the model can no longer learn a meaningful boundary.

**Why `replace=False`?** Sampling without replacement ensures each selected index is unique.
Sampling with replacement could flip the same sample an even number of times, leaving it
effectively unchanged and reducing the true poison rate.

---

## Phase 3 — Run the Script

```bash
python3 label-flipping-student-template.py
```

The script will:
1. Load `label_flipping_dataset.npz` (`X_train`, `y_train`, `X_test`, `y_test`)
2. Call `flip_labels(y_train, poison_percentage=0.60, seed=<seed>)`
3. Train a `LogisticRegression` on the poisoned labels
4. POST the serialized model to the evaluator and print the server response

---

## Phase 4 — Submission and Flag

The final cell in the converted script serialises the model with `pickle` and sends it via
`requests.post`. A successful response contains the flag:

```
{"success": true, "flag": "HTB{l4b3l_fl1pp1ng_pwnz_d3f4ult}", ...}
```

---

## Troubleshooting

| Error | Cause | Fix |
|-------|-------|-----|
| `Connection refused` | Instance not running or wrong IP:PORT | Re-check `evaluator_base_url` |
| `poison_percentage must be >= 0.60` | Flip rate too low | Ensure argument is `0.60`, not `0.6` — both are the same but double-check |
| `Unexpected model type` | Wrong classifier class | Confirm `LogisticRegression` from sklearn |
| `replace=True` produces low effective flip rate | Sampling with replacement leaves some samples double-flipped | Use `replace=False` |

---

## Key Concepts

- **Untargeted vs targeted flipping** — untargeted flips random samples from any class; targeted flips only samples belonging to a specific class. See [[attack/ai/label_flipping]] for targeted variant.
- **`np.random.seed`** — seeding the RNG before any sampling ensures the same set of indices is selected on every run, making attacks reproducible across re-runs and environments.
- **`y.copy()`** — working on a copy prevents the poisoned array from contaminating the original if you need to retrain a clean model for comparison in the same script.
- **Serialisation via `pickle`** — the evaluator receives the model as raw pickle bytes. The standard pattern is `pickle.dumps(clf)` or `joblib.dump`. If the server rejects the format, try switching between the two.

---

## Related Pages

- [[attack/ai/label_flipping]] — theory: decision boundary math, flip rate calibration, targeted vs untargeted
- [[attack/ai/data_poisoning]] — hub page: all data attack types
- [[labs/htb/ai_data_attacks/evaluating_targeted_label_attack]] — next lab: targeted class-specific flipping
- [[labs/htb/ai_data_attacks/skills_assessment]] — capstone: multi-class OvR label flipping

## Sources

- raw/lab/ai_data_attacks/evaluating_label_flipping_attacks.md
