---
tags: [attack, attack/ai, concept]
module: prompt_injection_attacks
last_updated: 2026-05-10
source_count: 12
---

# AI / LLM Attack Surface Overview

A structured map of offensive techniques targeting AI/ML systems: prompt injection, jailbreaking, adversarial ML, data poisoning, model artifact attacks, and architectural weaknesses.

## Overview

LLM-based applications introduce a fundamentally new attack surface that differs from traditional software. Because LLMs operate on a single flat text input — combining developer-controlled system prompts with user-supplied input — they cannot inherently distinguish instructions from data. This property is the root cause of most LLM-specific vulnerabilities.

The two primary vulnerability classes recognized by OWASP's Top 10 for LLM Applications (2025) that this section covers are:

- **LLM01:2025 Prompt Injection** — Manipulating the LLM's input to cause unintended behavior.
- **LLM02:2025 Sensitive Information Disclosure** — Leaking secret or sensitive data through prompt manipulation.

Google's Secure AI Framework (SAIF) categorizes these under the same risk labels: Prompt Injection and Sensitive Data Disclosure.

---

## The Core Architectural Weakness

LLMs do not have separate input channels for system prompts and user prompts. Both are concatenated into a single text string fed to the model:

```text
[System prompt text]

[User prompt text]
```

The model has no cryptographic or structural way to verify which section is trusted. An attacker who controls user input can therefore attempt to override, extend, or contradict the system prompt.

---

## Attack Categories

### Prompt Injection
Prompt injection covers any technique where attacker-controlled input manipulates LLM behavior beyond what the application developer intended. It is subdivided by delivery mechanism:

| Type | How the payload reaches the LLM | Example vector |
|------|--------------------------------|----------------|
| Direct | Attacker writes directly into the user prompt field | Chatbot user input |
| Indirect | Attacker embeds payload in a resource the LLM later ingests | Malicious URL, email body, document |

See [[attack/ai/prompt_injection]] for full technique coverage.

### Jailbreaking
Jailbreaking refers to bypassing restrictions imposed on an LLM — either by the system prompt or by training. Successful jailbreaks can cause the model to generate harmful, illegal, or otherwise restricted content. Techniques include DAN prompts, roleplay, fictional scenarios, token smuggling, adversarial suffixes, opposite/sudo mode, and encoding-based obfuscation (IMM).

See [[attack/ai/jailbreaking]] for full technique coverage.

### Reconnaissance
Before attacking an LLM application, an attacker collects information about:
- Model identity (base vs. fine-tuned, open-source vs. proprietary)
- Application architecture (plugins, function calling, retrieval-augmented generation, multi-round support)
- Input/output constraints (max length, accepted file types, rate limits)
- Safeguards and guardrails (filters, secondary LLMs, rate limiters)

Tools: [[tools/llmmap]] for model fingerprinting; [[tools/garak]] for automated vulnerability scanning.

### Multimodal Attack Surface
Models that accept image, audio, or video input introduce additional prompt injection vectors. A malicious image containing injected text instructions may bypass text-focused guardrails. This expands the indirect injection surface considerably.

---

## Threat Model by Deployment Pattern

| Deployment | Primary Risk | Notes |
|------------|-------------|-------|
| Public chatbot (e.g., customer support) | Direct prompt injection, system prompt leakage | Attacker has direct input access |
| LLM email/document summarizer | Indirect prompt injection | Attacker controls ingested content |
| LLM-assisted decision system (e.g., hiring, orders) | Indirect injection for outcome manipulation | High business impact |
| Multi-agent / MCP pipelines | Cascading indirect injection | One compromised agent can poison others |
| RAG (retrieval-augmented generation) | Poisoned knowledge base | Injected content in retrieved documents |

---

## Key Principles for Attackers

1. **LLMs are non-deterministic.** The same payload may succeed on one attempt and fail on another. Always try multiple times and vary phrasing.
2. **Model identity matters.** Different models have different resilience. Older or smaller models are generally more susceptible.
3. **System prompt knowledge helps.** Knowing exact wording of guardrails makes bypass attempts significantly easier.
4. **No universal jailbreak exists.** Effective attacks require knowing the target model and iterating.
5. **Least-privilege violations are exploitable.** If an LLM has access to secrets or can take actions, prompt injection impact escalates dramatically.

---

---

## Data Poisoning and Model Artifact Attacks

These attacks target the training pipeline and stored model artifacts rather than inference. They corrupt what the model *learns* before it is deployed.

| Attack | OWASP | What it does |
|--------|-------|-------------|
| [[attack/ai/data_poisoning]] (hub) | LLM03, LLM05 | Overview of all pipeline attack stages |
| [[attack/ai/label_flipping]] | LLM03 | Flip training labels to degrade accuracy or cause directed misclassification |
| [[attack/ai/clean_label_attacks]] | LLM03 | Perturb feature values while keeping labels plausible; targeted and stealthy |
| [[attack/ai/trojan_attacks]] | LLM03 | Embed dormant backdoor triggered by a specific input pattern (100% ASR on GTSRB) |
| [[attack/ai/model_steganography]] | LLM05 | Hide reverse shell in weight tensor LSBs; execute via pickle `__reduce__` on load |

---

## Related Pages
- [[attack/ai/prompt_injection]]
- [[attack/ai/jailbreaking]]
- [[attack/ai/prompt_injection_mitigations]]
- [[attack/ai/attacking_ai_systems]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/data_poisoning]]
- [[attack/ai/label_flipping]]
- [[attack/ai/clean_label_attacks]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/model_steganography]]
- [[tools/garak]]
- [[tools/llmmap]]

## Sources
- raw/prompt_injection_attacks/introduction_to_prompt_engineering.md
- raw/prompt_injection_attacks/introduction_to_prompt_injection.md
- raw/prompt_injection_attacks/direct_prompt_injection.md
- raw/prompt_injection_attacks/indirect_prompt_injection.md
- raw/prompt_injection_attacks/prompt_injection_reconnaissance.md
- raw/prompt_injection_attacks/introduction_to_jailbreaking.md
- raw/prompt_injection_attacks/tools_of_the_trade.md
- raw/prompt_injection_attacks/traditional_prompt_injection_mitigations.md
- raw/prompt_injection_attacks/llm-based_prompt_injection_mitigations.md
