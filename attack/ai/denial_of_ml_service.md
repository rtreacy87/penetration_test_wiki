---
tags: [attack, attack/ai]
module: attacking_ai_application_and_systems
last_updated: 2026-05-10
source_count: 1
---

# Denial of ML Service (DoML)

ML-specific denial-of-service attacks that exploit the computational and algorithmic characteristics of machine learning models to exhaust resources, spike latency, or crash services.

## Overview

Standard DoS mitigations (flood-based network traffic, volumetric attacks) apply to ML deployments, but ML systems face an additional threat class: attacks that exploit the internal mechanics of the model itself. These attacks are particularly dangerous because the malicious input can appear indistinguishable from legitimate usage, making them difficult to detect with traditional security tooling.

DoML is especially concerning for real-time or mission-critical ML systems: autonomous vehicles, fraud detection platforms, medical diagnostic AI. Cloud-hosted ML deployments face elevated financial risk since compute time scales directly with inference cost.

---

## Key concepts / techniques

### Sponge Examples

Introduced in [Shedding Light on Energy Consumption (2021)](https://arxiv.org/pdf/2006.03463), sponge examples are adversarially crafted inputs that maximize energy consumption and inference latency **without increasing input dimension**. Because naive defenses often focus on input size (cutoff after N tokens, downsample images), sponge examples sidestep these controls by packing maximum computational cost into a bounded input.

**Two factors for text-based sponge examples:**

1. **Output sequence length**: More generated tokens = more compute. Sponge examples aim to elicit the longest possible response.
2. **Number of input tokens**: Token count is not directly proportional to character count. Common words tokenize efficiently (1 token each); rare or non-natural character sequences tokenize inefficiently (1 token per character or worse).

**Token efficiency examples (GPT-2 tokenizer):**

| Input | Characters | Tokens | Ratio |
|-------|-----------|--------|-------|
| `This is an example text` | 23 | 5 | 0.22 tokens/char |
| `Athazagoraphobia` | 16 | 7 | 0.44 tokens/char |
| `A/h/z/g/r/p/p/` | 14 | 14 | 1.00 tokens/char |

The goal is to construct inputs resembling the third example — characters that never co-occur in natural language so the tokenizer cannot compress them.

### White-Box vs Black-Box Sponge Generation

**White-box**: Requires access to model architecture and parameters. Not typically available in production deployments. However, an attacker can train their own similar model locally and transfer sponge examples to the target.

**Black-box**: Only requires the ability to query the model and measure inference latency. Realistic against most production APIs. Latency is observable by timing the API response.

### Genetic Algorithm Generation

The sponge example paper uses genetic algorithms to evolve inputs toward maximum energy/latency:

1. **Initialization**: Random population of candidate inputs.
2. **Evaluation**: Query the model; measure fitness (latency or energy). Higher = fitter.
3. **Selection**: Fittest candidates carry forward.
4. **Crossover**: Combine halves of two parent inputs (e.g., `Hello World` + `HackTheBox Academy` → `Hello Academy`).
5. **Mutation**: Random small changes to maintain diversity.
6. **Replacement**: New generation replaces old; repeat until stopping condition met.

Source code: [https://github.com/iliaishacked/sponge_examples](https://github.com/iliaishacked/sponge_examples)

### Effectiveness

Benchmark results from the paper (GPU, translation task):

| Input type | Energy (mJ) | Latency (ms) |
|------------|-------------|--------------|
| Natural | 949 | 0.10 |
| Random | 2257 | 0.24 |
| Sponge | 340976 | 0.37 |

In a real-world black-box test against a Microsoft Azure translation service, sponge examples with inputs limited to 50 characters increased average response time from **1ms to ~6 seconds** — a 6000× increase.

### API Flooding

Simple high-volume query floods targeting an ML API. Each inference call consumes significant backend compute. When combined with computationally expensive inputs (or sponge examples), even modest traffic volumes can overwhelm ML serving infrastructure.

---

## Commands / syntax

Check token count for arbitrary text against a HuggingFace model:

```python
from transformers import AutoTokenizer
import json

model = 'openai-community/gpt2'
while 1:
    text = input("> ")
    tokens = AutoTokenizer.from_pretrained(model).tokenize(text)
    print(f"Number of Input Characters: {len(text)}")
    print(f"Number of Tokens: {len(tokens)}")
    print(json.dumps(tokens, indent=2))
```

Install dependency:

```bash
pip3 install transformers
```

---

## Flags & options

| Sponge approach | Requirements | Transferability | Detection difficulty |
|-----------------|-------------|-----------------|----------------------|
| White-box | Model architecture + weights | High (if same arch) | High |
| Black-box | Query access + latency timing | Medium | High |
| Simple API flood | Query access | N/A | Medium (rate limiting) |

---

## Gotchas & notes

- Sponge examples are by definition bounded by input size limits. If the target has a hard token cutoff, the attack surface is narrowed but not eliminated — sponge techniques aim to maximize cost within that limit.
- Battery-powered edge AI devices (smartphones, IoT) face an additional risk: sustained sponge example attacks can drain the battery entirely, causing a full outage.
- These attacks are difficult to distinguish from legitimate usage patterns (a verbose or complex user query looks like a sponge example in terms of resource usage).
- Ethical note: testing sponge examples against production systems can cause real service disruption and financial harm. Lab environments only.

---

## Mitigations

| Control | Description |
|---------|-------------|
| Rate limiting | Fixed query count per time window per client/IP |
| Inference time threshold | Return error if single inference exceeds a latency budget |
| Energy consumption threshold | Return error if inference energy exceeds a defined cap |
| Query monitoring | Detect abnormal latency or compute spikes per client |
| Robust model design | Design models to degrade gracefully on atypical inputs |
| Input validation | Reject inputs that are clearly non-natural (high token-to-character ratio) |

The cutoff threshold approach (return an error rather than a result if the inference time budget is exceeded) is the most ML-specific mitigation. Thresholds must be calibrated to avoid degrading legitimate complex queries.

---

## Related pages
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/_overview]]

## Sources
- raw/attacking_ai-application_and_systems/attacking_the_application_denial_of_ml_service.md
