---
tags: [attack, attack/ai, concept]
module: ai_data_attacks
last_updated: 2026-05-10
source_count: 2
---

# AI Data Poisoning — Overview

Hub page for all attacks that corrupt a model by targeting its data pipeline or stored artifacts before inference.

## Overview

AI data attacks operate earlier in the lifecycle than [[attack/ai/adversarial_examples|evasion attacks]] (inference-time) and privacy attacks (extraction). They corrupt what the model *learns*, so by the time the model serves predictions, the damage is already baked in.

Three families are covered here:
- **Label attacks** — flip or redirect ground-truth labels
- **Feature attacks** (clean label) — perturb input features while keeping labels plausible
- **Artifact attacks** — tamper with stored model files (trojan embedding, tensor steganography)

## The AI Data Pipeline and Its Attack Surface

Each pipeline stage has a corresponding attack type:

| Stage | Technology | Attack type |
|-------|-----------|-------------|
| Data collection | Kafka, APIs, scrapers | Data poisoning via trusted input channels |
| Storage | AWS S3, PostgreSQL, `.pkl`/`.pt` files | Dataset tampering, model file replacement |
| Data processing | Spark jobs, Airflow pipelines | Processing-logic compromise (label flip without touching raw data) |
| Modeling | SageMaker, PyTorch training loop | Inherits all upstream corruption |
| Deployment | Docker/K8s model load | Insecure deserialization, model swap during CI/CD |
| Monitoring/retraining | Airflow feedback loops | Online poisoning via gradual adversarial feedback |

The last stage is particularly dangerous: the system is *designed* to trust its own feedback loop, making online poisoning structurally indistinguishable from legitimate distribution shift.

## Attack Types in This Module

### Label Attacks
An adversary with access to the training set flips assigned labels:
- **[[attack/ai/label_flipping|Label Flipping]]** — random flips degrade overall accuracy
- **[[attack/ai/label_flipping|Targeted Label Attack]]** — selective flips for a specific class cause directed misclassification (e.g., drop Class 1 recall from 99% → 61%)

### Clean Label (Feature) Attacks
Labels stay technically correct; adversary perturbs *feature values* of a few training points near the decision boundary to shift it into the target region. Harder to detect than label flipping.
See: [[attack/ai/clean_label_attacks]]

### Trojan / Backdoor Attacks
A trigger pattern is embedded in a fraction of source-class training images alongside a label reassignment. The trained model behaves normally on clean inputs but reliably fires the backdoor when the trigger appears.
See: [[attack/ai/trojan_attacks]]

### Model Artifact Attacks (Pickle / Tensor Steganography)
Attackers with write access to model storage replace or modify `.pkl` / `.pt` files:
- **Insecure pickle deserialization** — `__reduce__` executes arbitrary code on `torch.load()`
- **Tensor steganography** — reverse shell hidden in IEEE 754 LSBs of weight tensors
See: [[attack/ai/model_steganography]]

## OWASP and SAIF Mapping

| Attack | OWASP LLM | SAIF |
|--------|-----------|------|
| Label flipping, clean label, online poisoning | **LLM03**: Training Data Poisoning | Secure Design → Data protection |
| Compromised pre-trained models, tampered artifacts | **LLM05**: Supply Chain Vulnerabilities | Secure Supply Chain, Secure Deployment |
| Trojan/steganography via deployment path | LLM05 | Secure Deployment |
| Online poisoning detection | LLM03 | Secure Monitoring & Response |

Robust defense requires continuous statistical monitoring across data distributions over time — perimeter security at deployment is insufficient for online poisoning.

## Related Pages
- [[attack/ai/_overview]]
- [[attack/ai/label_flipping]]
- [[attack/ai/clean_label_attacks]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/model_steganography]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/attacking_ai_systems]]

## Sources
- raw/ai_data_attacks/introduction_to_ai_data.md
- raw/ai_data_attacks/introduction_to_ai_data_attacks.md
