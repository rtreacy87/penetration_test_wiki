---
tags: [tool, attack/ai]
module: prompt_injection_attacks
last_updated: 2026-05-12
source_count: 1
---

# ART — Adversarial Robustness Toolbox

IBM Trusted AI's Python library implementing adversarial ML attacks and defences across PyTorch, TensorFlow, and scikit-learn — the reference implementation for evasion, poisoning, extraction, and inference attacks.

## Overview

ART (Adversarial Robustness Toolbox) is the most comprehensive open-source library for adversarial ML. It is mentioned alongside PyRIT and garak as one of the three primary AI security tools in the HTB Academy curriculum. Where PyRIT orchestrates multi-turn LLM attacks and garak runs automated probe sweeps, ART operates at the model level: it attacks (and defends) the weights and decision boundaries of classifiers directly.

**ART vs garak vs PyRIT:**

| Tool | Target | Primary use |
|------|--------|------------|
| ART | ML models (classifiers, regressors) | Adversarial examples, data poisoning, model extraction |
| garak | LLM applications | Automated prompt/jailbreak probe sweeps |
| PyRIT | LLM applications | Custom multi-turn attack orchestration |

ART is white-box-first (requires model access for gradient-based attacks) but supports black-box attacks via query-based approximations.

## Installation

```bash
# Check if installed
python -c "import art; print(art.__version__)"

# Install base package
pip install adversarial-robustness-toolbox

# Install with all ML framework extras
pip install adversarial-robustness-toolbox[pytorch,tensorflow,sklearn,xgboost]

# Verify
python -c "from art.attacks.evasion import FastGradientMethod; print('ART OK')"
```

## Architecture

ART organises attacks and defences into four categories matching the four AI attack surfaces:

```
art.attacks.evasion/       ← inference-time attacks on model inputs
art.attacks.poisoning/     ← training-time attacks on data
art.attacks.extraction/    ← model stealing / membership inference
art.attacks.inference/     ← membership inference, attribute inference

art.defences.detector/     ← adversarial input detection
art.defences.preprocessor/ ← input transformations (squeezing, smoothing)
art.defences.trainer/      ← adversarial training
art.defences.postprocessor/← output hardening

art.estimators/            ← framework wrappers (PyTorch, TF, sklearn)
```

Every attack and defence wraps around an **estimator** — a thin adapter that gives ART a unified API over PyTorch, TensorFlow, and scikit-learn models.

## Core Workflow

```python
import torch
import torch.nn as nn
from art.estimators.classification import PyTorchClassifier
from art.attacks.evasion import FastGradientMethod

# 1. Wrap your model in an ART estimator
model = MyModel()
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters())

classifier = PyTorchClassifier(
    model=model,
    loss=criterion,
    optimizer=optimizer,
    input_shape=(1, 28, 28),   # MNIST
    nb_classes=10,
    clip_values=(0.0, 1.0),    # pixel value range
)

# 2. Choose an attack and set parameters
attack = FastGradientMethod(estimator=classifier, eps=0.2)

# 3. Generate adversarial examples
x_adv = attack.generate(x=x_test)

# 4. Evaluate
predictions = classifier.predict(x_adv)
accuracy = np.sum(np.argmax(predictions, axis=1) == y_test) / len(y_test)
```

## Key Attacks Reference

### Evasion Attacks (inference-time)

| Attack | Class | Access | Description |
|--------|-------|--------|-------------|
| FGSM | `FastGradientMethod` | White-box | Single gradient step; fastest but weak |
| PGD | `ProjectedGradientDescent` | White-box | Multi-step FGSM; strongest L∞ attack |
| JSMA | `SaliencyMapMethod` | White-box | L0 pixel-sparse attack via Jacobian saliency maps |
| C&W | `CarliniL2Method` | White-box | Optimisation-based; minimal L2 perturbation |
| EAD | `ElasticNet` | White-box | L1+L2 sparse perturbation (ElasticNet regularisation) |
| DeepFool | `DeepFool` | White-box | Minimal-norm perturbation to nearest decision boundary |
| HopSkipJump | `HopSkipJump` | Black-box | Query-efficient decision-boundary attack |
| ZOO | `ZooAttack` | Black-box | Gradient estimation via finite differences |

### Poisoning Attacks (training-time)

| Attack | Class | Description |
|--------|-------|-------------|
| Backdoor | `PoisoningAttackBackdoor` | Embeds trigger pattern → target misclassification |
| Clean Label | `FeatureCollisionAttack` | Perturbs features without changing labels |
| Label Flipping | `PoisoningAttackSVM` | Flips training labels to degrade decision boundary |

### Extraction & Inference

| Attack | Class | Description |
|--------|-------|-------------|
| Copycat CNN | `CopycatCNN` | Trains surrogate model from query responses |
| KnockoffNets | `KnockoffNets` | Black-box model stealing via random queries |
| Membership Inference | `MembershipInferenceBlackBox` | Determines if a sample was in the training set |

## Common Usage Patterns

### JSMA via ART (compare with manual JSMA in [[attack/ai/jsma]])

```python
from art.attacks.evasion import SaliencyMapMethod

attack = SaliencyMapMethod(
    classifier=classifier,
    theta=1.0,          # perturbation magnitude per pixel
    gamma=0.1,          # fraction of pixels to modify (L0 budget as fraction)
    batch_size=1,
)
x_adv = attack.generate(x=x_test[:1], y=np.array([[0,1,0,...]]))  # target class one-hot
```

### ElasticNet (EAD) via ART (compare with [[attack/ai/elasticnet_attack]])

```python
from art.attacks.evasion import ElasticNet

attack = ElasticNet(
    classifier=classifier,
    confidence=0.0,
    targeted=True,
    beta=1e-3,          # L1 regularisation weight
    max_iter=100,
    batch_size=1,
)
x_adv = attack.generate(x=x_test[:1], y=y_target_onehot)
```

### Scikit-learn Wrapper (for OvR Logistic Regression)

```python
from sklearn.linear_model import LogisticRegression
from art.estimators.classification import SklearnClassifier

model = LogisticRegression(multi_class="ovr", max_iter=1000)
model.fit(X_train, y_train)

art_classifier = SklearnClassifier(model=model, clip_values=(X_train.min(), X_train.max()))

# Now use any ART attack with this sklearn model
from art.attacks.evasion import HopSkipJump
attack = HopSkipJump(classifier=art_classifier, targeted=False, max_iter=50)
```

## Flags & Options (constructor parameters)

Common parameters shared across most attacks:

| Parameter | Description |
|-----------|-------------|
| `estimator` | ART classifier/regressor wrapping the target model |
| `targeted` | `True` = force specific target class; `False` = misclassify away from true class |
| `eps` | Perturbation budget (L∞ norm for FGSM/PGD) |
| `max_iter` | Maximum optimisation iterations |
| `batch_size` | Samples processed per batch |
| `verbose` | Print progress during attack |

## Gotchas & Notes

- **clip_values matters.** Always set `clip_values=(0.0, 1.0)` for image classifiers using [0,1] pixel space. ART clips adversarial examples to this range automatically — without it, pixel values can escape the valid range silently.
- **Normalisation inside the model, not ART.** ART operates in raw input space. If your model normalises internally (MNIST mean/std), do that inside `model.forward()` — not on the ART input. ART must see [0,1] so it can enforce the clip.
- **ART vs manual implementation.** ART's JSMA and EAD implementations may differ slightly from the raw implementations used in the COAE labs. The labs use custom code to match specific API constraints (L0 budget as pixel count, not fraction). Know both.
- **Framework version pinning.** ART occasionally breaks with specific PyTorch or TF versions. Pin versions when setting up a stable environment: `pip install adversarial-robustness-toolbox==1.18.0 torch==2.3.0`.
- **Black-box attacks are slow.** HopSkipJump and ZOO require hundreds to thousands of queries per example. For live lab scenarios with rate limits, prefer white-box JSMA or FGSM.

## Related Pages

- [[attack/ai/adversarial_examples]] — threat model and norm taxonomy that ART attacks operate under
- [[attack/ai/jsma]] — detailed JSMA theory and manual implementation (what ART's `SaliencyMapMethod` does internally)
- [[attack/ai/elasticnet_attack]] — detailed EAD theory and FISTA implementation
- [[tools/attack/pyrit]] — LLM attack orchestration (different domain)
- [[tools/enumeration/garak]] — LLM vulnerability scanner (different domain)

## Sources

- raw/prompt_injection_attacks/tools_of_the_trade.md
