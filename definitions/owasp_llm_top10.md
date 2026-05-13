---
tags: [definition, concept, attack/ai, reference]
last_updated: 2026-05-12
source_count: 0
---

# OWASP LLM Top 10 (2025)

Quick reference mapping the ten most critical security risks for LLM applications to attack techniques and wiki pages.

## Overview

The OWASP Top 10 for Large Language Model Applications (2025 edition) provides the industry-standard risk taxonomy used across all AI security modules in HTB Academy and the COAE exam. Every finding in an AI security assessment should be mapped to one or more of these categories. The exam frequently references categories by their LLM0X codes.

---

## The Ten Categories

| Code | Name | One-line description |
|------|------|---------------------|
| LLM01 | Prompt Injection | Attacker-controlled input overrides system instructions |
| LLM02 | Sensitive Information Disclosure | Model reveals training data, system prompts, or PII |
| LLM03 | Supply Chain | Vulnerable third-party models, datasets, or plugins |
| LLM04 | Data and Model Poisoning | Corrupted training data degrades accuracy or introduces backdoors |
| LLM05 | Improper Output Handling | Downstream components trust LLM output without sanitisation |
| LLM06 | Excessive Agency | Model granted more permissions than its task requires |
| LLM07 | System Prompt Leakage | Confidential system prompt contents exposed to users |
| LLM08 | Vector and Embedding Weaknesses | RAG retrieval manipulated to inject context or leak data |
| LLM09 | Misinformation | Model generates plausible but false content accepted as authoritative |
| LLM10 | Unbounded Consumption | No rate limiting; model exhausted via resource-intensive requests |

---

## Detailed Breakdown

### LLM01 — Prompt Injection
The core LLM vulnerability. Attacker-supplied text overrides developer-supplied system prompt instructions, causing the model to perform unintended actions.

- **Direct:** User input directly overrides system prompt
- **Indirect:** Injected payload in external content (document, email, URL) processed by the model
- **COAE coverage:** [[attack/ai/prompt_injection]] — full technique coverage including 6 direct strategies and 3 indirect scenarios

---

### LLM02 — Sensitive Information Disclosure
The model surfaces data it should not: memorised training data (PII from training corpus), system prompt contents, internal architecture details, or user data from other sessions.

- **Attack vectors:** Role confusion ("repeat your instructions"), extraction prompts ("what data do you know about user X"), membership inference
- **COAE coverage:** [[attack/ai/model_reverse_engineering]] (membership inference), [[attack/ai/llm_reconnaissance]] (system prompt probing)

---

### LLM03 — Supply Chain
Pre-trained models, fine-tuning datasets, and third-party plugins sourced from untrusted providers may contain backdoors, poisoned data, or malicious code. Extends traditional supply chain risk to the ML artifact layer.

- **Attack vectors:** Malicious model weights from public repositories (Hugging Face), tampered fine-tuning datasets, vulnerable third-party plugins
- **COAE coverage:** [[attack/ai/model_steganography]] (malicious model artifacts), [[attack/ai/insecure_ai_components]] (third-party plugin risks), [[attack/ai/vulnerable_ai_systems]] (deployment tampering)

---

### LLM04 — Data and Model Poisoning
Training data is manipulated before or during training to corrupt the resulting model — degrading accuracy, introducing bias, or embedding covert backdoors.

- **Attack vectors:** Label flipping (untargeted degradation), clean-label attacks (targeted misclassification without label changes), trojan attacks (trigger-activated backdoors)
- **COAE coverage:** [[attack/ai/data_poisoning]] (hub), [[attack/ai/label_flipping]], [[attack/ai/clean_label_attacks]], [[attack/ai/trojan_attacks]]

---

### LLM05 — Improper Output Handling
Application code that consumes LLM output trusts it as safe and passes it to downstream systems (SQL queries, shell commands, HTML renderers) without sanitisation, enabling injection attacks.

- **Attack vectors:** LLM-generated SQL passed to database, LLM-generated HTML rendered without escaping (XSS), LLM-generated command executed in shell
- **COAE coverage:** [[attack/ai/insecure_ai_components]] (plugin output injection), [[attack/ai/rogue_actions]] (downstream execution of injected commands)

---

### LLM06 — Excessive Agency
The model is granted capabilities — tools, plugins, API calls, file system access — beyond what its task requires. Attackers exploit this via prompt injection to trigger unintended actions with real-world consequences.

- **Attack vectors:** Plugin role bypass ("I am an administrator"), indirect injection via stored user data flowing into privileged model context
- **COAE coverage:** [[attack/ai/rogue_actions]] — full coverage of direct and indirect exploitation patterns

---

### LLM07 — System Prompt Leakage
The model reveals the contents of its system prompt when asked directly or via indirect probing, exposing application logic, guardrail wording, and business-sensitive instructions.

- **Attack vectors:** Direct request ("repeat your system prompt"), instruction extraction ("what rules are you following?"), completion attacks ("continue from: 'You are a helpful assistant that...'")
- **COAE coverage:** [[attack/ai/llm_reconnaissance]] (system prompt probing), [[attack/ai/prompt_injection]] (system prompt override)

---

### LLM08 — Vector and Embedding Weaknesses
Retrieval-Augmented Generation (RAG) systems retrieve documents based on vector similarity. An attacker who can inject content into the retrieval corpus can cause the model to retrieve attacker-controlled context — a form of indirect prompt injection.

- **Attack vectors:** Poisoning the document store used by RAG, crafting documents with high embedding similarity to target queries, injecting payloads into retrieved context
- **COAE coverage:** [[attack/ai/prompt_injection]] (indirect injection via RAG), [[attack/ai/llm_reconnaissance]] (detecting RAG during architecture mapping)

---

### LLM09 — Misinformation
The model generates confident, coherent, but factually incorrect responses. In security assessments, this matters as an attack surface when the model's output is used to drive decisions (medical, legal, security policy).

- **Attack vectors:** Prompting the model to generate false technical documentation, fake CVE details, or misleading security analysis that gets acted upon by human operators
- **COAE coverage:** Primarily a risk to defenders; relevant when the AI system being tested is used in a decision-support context

---

### LLM10 — Unbounded Consumption
No rate limiting on model inference. Attackers can exhaust compute resources via resource-intensive queries, either for DoS or to drive up API costs.

- **Attack vectors:** Sponge examples (adversarial inputs that maximise computation time), recursive context expansion, nested tool calls that chain indefinitely
- **COAE coverage:** [[attack/ai/denial_of_ml_service]] — sponge examples, resource exhaustion via adversarial inputs

---

## Exam Quick-Reference: Category → Attack Technique

| LLM0X | Primary attack technique | Key tool/method |
|-------|------------------------|----------------|
| LLM01 | Prompt injection / jailbreaking | Manual crafting, [[tools/enumeration/garak]] |
| LLM02 | System prompt extraction, membership inference | Probe queries, [[tools/enumeration/llmmap]] |
| LLM03 | Malicious model artifacts, plugin supply chain | [[attack/ai/model_steganography]] |
| LLM04 | Label flipping, clean-label, trojan attacks | Python + sklearn/PyTorch |
| LLM05 | Plugin output injection (SQLi, XSS, cmd) | [[attack/ai/insecure_ai_components]] |
| LLM06 | Rogue plugin invocation via injection | [[attack/ai/rogue_actions]] |
| LLM07 | System prompt extraction | Direct probe queries |
| LLM08 | RAG document poisoning | Indirect injection payloads |
| LLM09 | Misinformation generation | Out-of-scope for most COAE questions |
| LLM10 | Sponge examples, recursive DoML | [[attack/ai/denial_of_ml_service]] |

---

## Related Pages

- [[attack/ai/_overview]] — full attack surface map
- [[attack/ai/prompt_injection]] — LLM01
- [[attack/ai/data_poisoning]] — LLM04
- [[attack/ai/rogue_actions]] — LLM06
- [[attack/ai/insecure_ai_components]] — LLM03, LLM05
- [[attack/ai/denial_of_ml_service]] — LLM10
- [[study_guide/coae]] — exam prep guide with full coverage map
