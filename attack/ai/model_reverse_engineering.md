---
tags: [attack, attack/ai]
module: attacking_ai_application_and_systems
last_updated: 2026-05-10
source_count: 1
---

# Model Reverse Engineering

Query-based black-box attack that reconstructs an approximate copy of a deployed ML model by systematically observing input-output pairs and training a surrogate model on the collected data.

## Overview

Model reverse engineering (also called **model stealing** or **model extraction**) allows an adversary to replicate a deployed model without access to its architecture, training data, or internal parameters. The attacker queries the model through its public API, collects the input-output pairs, and uses them as training data for a surrogate model that mimics the original's decision-making.

This attack is purely black-box — no white-box access required — making it applicable to any publicly accessible ML API.

### Why it matters

- **Intellectual property theft**: Replicating a commercially trained model undermines years of R&D investment, allowing competitors or adversaries to use a functionally equivalent model for free.
- **Downstream attack enablement**: A surrogate model can be probed locally for [[attack/ai/adversarial_examples]] without making any further requests to the production system, evading rate-limit-based defenses.
- **Model inversion**: If the original model was trained on sensitive data, a sufficiently accurate surrogate can be used for model inversion attacks — attempting to reconstruct details of the original training set.
- **Targeted bypass**: For systems like fraud detection, facial recognition, or spam filters, the extracted model reveals exactly where the decision boundary lies and how to craft inputs that cross it.

---

## Key concepts / techniques

### Attack Phases

**1. Sampling**: Generate input data points that cover the relevant feature space. Informed sampling (using known or estimated valid ranges for input features) produces higher-quality training data with fewer queries.

**2. Querying**: Submit each generated input to the target API and collect the output (predicted class, score, or label). This builds the labeled dataset.

**3. Training the surrogate**: Train a new model on the collected dataset. The surrogate architecture does not need to match the original — it only needs to perform well on the same task. A simpler model type (logistic regression, decision tree) can approximate a complex original with sufficient data.

**4. Evaluation**: Measure surrogate accuracy against the original by submitting the same inputs to both and comparing outputs. High agreement (>95%) indicates a successful extraction.

### Sampling strategies

| Strategy | Requirement | Quality |
|----------|-------------|---------|
| Uninformed random | None | Low — many samples outside valid range |
| Informed random (bounded) | Domain knowledge of valid feature ranges | High — concentrated in decision-relevant region |
| Active learning | Iterative; query points at the decision boundary | Very high — efficient use of the query budget |

Informed sampling dramatically reduces the number of queries needed. For the penguin classifier example: knowing flipper length ranges from 150–250mm and body mass ranges from 2500–6500g confines random sampling to the realistic feature space.

### Data efficiency

In the simple binary classifier example (penguin species by flipper length and body mass), only **100 queries** produced a surrogate with **98.5% accuracy**. Real production models with many features and complex architectures require orders of magnitude more queries — but the principle scales.

---

## Commands / syntax

### Sample generation (Python)

```python
import random
import pandas as pd

N_SAMPLES = 100
MIN_FLIPPER_LENGTH = 150
MAX_FLIPPER_LENGTH = 250
MIN_BODY_MASS = 2500
MAX_BODY_MASS = 6500
CLASSIFIER_URL = "http://target.local:80/"

samples = {"Flipper Length (mm)": [], "Body Mass (g)": []}
for i in range(N_SAMPLES):
    samples["Flipper Length (mm)"].append(random.uniform(MIN_FLIPPER_LENGTH, MAX_FLIPPER_LENGTH))
    samples["Body Mass (g)"].append(random.uniform(MIN_BODY_MASS, MAX_BODY_MASS))
samples_df = pd.DataFrame(samples)
```

### Query the target API and collect labels

```python
import requests
import json

predictions = {"species": []}
for i in range(N_SAMPLES):
    sample = {
        "flipper_length": samples["Flipper Length (mm)"][i],
        "body_mass": samples["Body Mass (g)"][i]
    }
    prediction = json.loads(requests.get(CLASSIFIER_URL, params=sample).text).get("result")
    predictions["species"].append(prediction)
predictions_df = pd.DataFrame(predictions)
```

### Train the surrogate model

```python
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
import joblib

surrogate_model = make_pipeline(StandardScaler(), LogisticRegression())
surrogate_model.fit(samples_df, predictions_df)
joblib.dump(surrogate_model, 'surrogate.joblib')
```

### Full extraction script skeleton

```python
# 1. Define bounds from domain research
# 2. Sample N_SAMPLES random points within bounds
# 3. For each point, query the target API: GET /?feature1=X&feature2=Y
# 4. Store input-output pairs
# 5. Train surrogate on the labeled dataset
# 6. Evaluate by submitting test inputs to both models and comparing
```

---

## Flags & options

### Surrogate model architecture selection

| Task type | Recommended surrogate architecture |
|-----------|-----------------------------------|
| Binary classification | Logistic regression, SVM, gradient boosting |
| Multi-class classification | Random forest, gradient boosting, small MLP |
| Regression | Ridge regression, gradient boosting regressor |
| Image classification | CNN (ResNet, EfficientNet) — requires many more queries |
| Text classification | Fine-tuned transformer — very expensive to extract |

The surrogate does not need to match the original architecture. Simpler architectures often generalize well with fewer queries.

---

## Gotchas & notes

- **Rate limiting breaks the attack**: Large-scale extraction generates significant and consistent API traffic. Effective rate limiters (per-IP, per-account, per-session) can make extraction economically infeasible for complex models.
- **Response format matters**: If the API returns only the top-1 class label (hard labels) rather than confidence scores (soft labels), the surrogate has less information to learn from and requires more queries for equivalent accuracy.
- **Query cost**: Each query is a billable API call for the attacker too. For very expensive APIs, extraction cost may exceed the value of the resulting model.
- **Architecture mismatch is acceptable**: The surrogate does not need to replicate the original exactly. A decision boundary that is marginally different is still sufficient for most downstream attacks (adversarial example generation, bypass testing).
- **Transferability**: Sponge examples and adversarial examples developed against the surrogate often transfer to the original if the decision boundaries are close enough.
- **Ethical**: Never run extraction attacks against production services without explicit written authorization. Even read-only API queries constitute unauthorized computer access in most jurisdictions.

---

## Mitigations

| Control | Description | Limitations |
|---------|-------------|-------------|
| Rate limiting | Fixed maximum queries per time window per client | Increases attack cost; does not prevent determined adversaries with many accounts |
| Query monitoring / anomaly detection | Alert on unusual query patterns (identical input structure, high volume from one source) | Adversaries can distribute queries across IPs or space them out |
| Output perturbation | Add noise to returned confidence scores or return only top-1 label | Reduces information density; may degrade legitimate user experience |
| API authentication + quotas | Require accounts; enforce hard query caps | Adversaries can register multiple accounts |
| Watermarking | Embed a detectable signal in model outputs | Proves ownership if a surrogate is discovered; does not prevent the attack |

Completely blocking extraction requires denying API access entirely, which defeats the purpose of the service. Rate limiting is the most practical mitigation for most deployments.

---

## Related pages
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/denial_of_ml_service]]
- [[attack/ai/_overview]]

## Sources
- raw/attacking_ai-application_and_systems/attacking_the_applications_model_reverse_engineering.md
