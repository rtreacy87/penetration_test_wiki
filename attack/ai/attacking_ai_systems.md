---
tags: [attack, attack/ai]
module: attacking_ai_application_and_systems
last_updated: 2026-05-10
source_count: 8
---

# Attacking AI Systems

Hub page covering the two primary layers of attack against real-world AI deployments: the **application layer** (what users interact with) and the **system layer** (the infrastructure that runs it).

## Overview

Real-world AI deployments consist of four distinct components: Model, Data, Application, and System. This page focuses on the application and system components. Earlier attack modules (prompt injection, LLM output attacks, data attacks) cover the model and data layers — the attacks catalogued here target the surrounding infrastructure.

The application component serves as the interface layer connecting users to the underlying model: web apps, mobile apps, APIs, databases, plugins, and autonomous agents. The system component encompasses the infrastructure layer: deployment platforms, source code, data storage, model serving frameworks, and the hardware they run on.

Vulnerabilities in either layer can cascade across the entire deployment. A web application SQLi can expose all LLM conversation history. A misconfigured model serving framework can yield full remote code execution.

---

## Application Layer Attacks

### Denial of ML Service (DoML)

Attackers flood an ML API with computationally expensive queries, or craft adversarial inputs that trigger worst-case behavior in the model (long inference paths, inefficient internal computation). Unlike traditional DoS, these attacks can look like ordinary usage.

See [[attack/ai/denial_of_ml_service]] for full detail.

### Insecure Integrated Components

Real-world ML apps are composed of many interacting components. Vulnerabilities in any integrated piece — web applications, plugins, APIs — affect the entire ML deployment.

**Web application vulnerabilities** directly impact the LLM pipeline:
- **IDOR**: LLM conversation logs stored with integer IDs (`/query/5`) may be enumerable if access control is missing. Fuzz with `ffuf` or `seq`.
- **SQL injection**: If conversation IDs are used in database queries, a single quote on the URL parameter may expose the backend (`/query/5'`). UNION-based payloads can exfiltrate the entire LLM interaction history.

**Plugin vulnerabilities**: LLM plugins often lack the access controls the main web application implements.
- A plugin providing conversation summaries by ID may not enforce that the requesting user owns the conversation — allowing IDOR through the chatbot interface.
- If authorization is enforced by a parameter the LLM passes to the plugin (rather than server-side from HTTP context), [[attack/ai/prompt_injection]] can bypass it by tricking the model into supplying a different user ID.
- Plugin output that is not sanitized before further processing can lead to secondary injection vulnerabilities (SQLi, command injection) via crafted LLM responses.

**Mitigations**: Third-party plugins from trusted, security-audited sources only. Least privilege on all plugins. Input from users and output from models treated as untrusted at every step. Defense-in-depth: rate limiting, monitoring, sandboxing.

### Model Reverse Engineering

Systematic query-based extraction of a deployed model's behavior, used to train a surrogate model that approximates the original without access to training data or model internals.

See [[attack/ai/model_reverse_engineering]] for full detail.

### Rogue Actions

Unintended actions executed by an AI agent via its plugins or extensions. Can result from:
- **Excessive agency**: The model has more capability than needed for its task (e.g., an arbitrary SQL query plugin accessible to a customer-facing chatbot).
- **Prompt injection exploit**: An attacker tricks the LLM into calling a privileged plugin by bypassing LLM-enforced access control (e.g., claiming to be an administrator).
- **Indirect prompt injection**: An attacker embeds a prompt injection payload in data that will later be fetched by a privileged context (e.g., a username containing an injection payload that an admin chatbot will process, triggering a `SQLQuery` plugin call in that elevated context).

**Attack chain (indirect)**:
1. Register a username containing a prompt injection payload targeting a privileged plugin.
2. Place an order to create a traceable record.
3. Wait for an administrator's chatbot (which has access to the privileged plugin) to query the order.
4. The order status response includes the malicious username, which the LLM processes and acts upon.

**Mitigations**: Strict plugin permission frameworks (least privilege, explicit capability declarations). Confirmation prompts before sensitive actions. Runtime auditing. Do not enforce access control solely through LLM instructions.

---

## System Layer Attacks

### Excessive Data Handling and Insecure Storage

ML applications process large volumes of potentially sensitive data. When combined with weak storage security, excessive data handling dramatically increases breach impact.

**Attack angle**: Beyond assessing the ML model itself, apply standard web application reconnaissance. Directory brute-forcing can reveal exposed database files:

```bash
gobuster dir -u http://<TARGET>/ -w /opt/SecLists/Discovery/Web-Content/raft-small-words.txt -x .db,.txt,.html
```

A publicly accessible `storage.db` file can contain every LLM conversation log, including any sensitive data users entered (credit card numbers, medical information). Common web vulnerabilities like SQL injection in the application frontend can achieve the same data exfiltration in more hardened environments.

**Mitigations**: Principle of data minimization (collect only what is required for the ML task). Encryption at rest. Access control on data stores. Data retention policies. Consider Homomorphic Encryption for highest-sensitivity use cases.

See [[attack/ai/attacking_ai_systems]] for context. Detailed page: see notes below on data handling (covered in the system attack sources).

### Model Deployment Tampering

Attackers manipulate the model or its training data through the deployment pipeline:
- **Direct**: Exploit broken access control to upload a modified model file, directly altering weights and biases.
- **Indirect (data poisoning)**: Gain access to training data (e.g., via a misconfigured FTP server) and poison it to introduce backdoors or biases. The poisoned model passes standard evaluation but fails in targeted, malicious ways under specific trigger conditions.

**Real-world example — ShellTorch (TorchServe)**:

A three-vulnerability chain:
1. Management API exposed on all interfaces, no authentication required (misconfiguration).
2. SSRF via the `/workflows` endpoint — no URL validation on the `url` GET parameter (CVE-2023-43654).
3. Deserialization RCE via a vulnerable version of the Java library SnakeYaml (CVE-2022-1471).

Exploit chain: access the management API → use SSRF to cause TorchServe to fetch a malicious `.war` archive from attacker infrastructure → the archive contains a SnakeYaml gadget that loads attacker-controlled Java code → RCE.

**Mitigations**: Verify third-party dependencies; keep them patched. Secure, hardened CI/CD pipelines. MFA on model management interfaces. Regular integrity checks. Reproducible builds. Automated vulnerability scanning.

### Vulnerable Framework Code (Supply Chain)

ML frameworks are frequent targets of CVEs that expose the entire ML pipeline. Examples:

| CVE | Package | Type | Versions |
|-----|---------|------|----------|
| CVE-2025-1975 | ollama | DoS (array bounds crash) | 0.5.11 |
| CVE-2023-6909 | MLflow | LFI / path traversal (query string) | 2.7.1 |
| CVE-2024-1594 | MLflow | LFI / path traversal (URL fragment bypass of prior fix) | 2.9.2 |

**CVE-2023-6909 (MLflow LFI)**: Create an experiment with a crafted `artifact_location` containing `?/../../../../../../../../../` to traverse to the filesystem root, then link a registered model to a run with `source: file:///` to download arbitrary files via `/model-versions/get-artifact?path=etc/passwd`.

**CVE-2024-1594**: The fix for CVE-2023-6909 only blocked `..` in query strings. Moving the traversal sequence to a URL fragment (`#../../../../../../../../../etc/`) bypasses the check entirely.

**Mitigations**: Audit ML framework versions against public CVE databases. Apply security patches promptly. Pin dependency versions and verify checksums.

---

## MCP (Model Context Protocol)

MCP introduces a standardized attack surface between LLM applications and external tools. Attacks include malicious MCP servers (tool poisoning, rug pull, tool shadowing), vulnerable MCP server implementations (SQLi, command injection, SSRF, information disclosure), and prompt injection via tool descriptions or tool results.

Full coverage: [[attack/ai/mcp_security]]

---

## Attack Surface Summary

| Layer | Attack | Primary Impact |
|-------|--------|----------------|
| Application | DoML (sponge examples, API flooding) | Availability |
| Application | Insecure web app components (IDOR, SQLi) | Confidentiality |
| Application | Insecure plugin components (IDOR, injection) | Confidentiality, integrity |
| Application | Model reverse engineering | IP theft, secondary attacks |
| Application | Rogue actions (direct/indirect) | Integrity, confidentiality |
| System | Insecure data storage | Confidentiality |
| System | Model deployment tampering (direct/supply chain) | Integrity, availability |
| System | Vulnerable framework code (CVEs) | Full system compromise |

---

## Related pages
- [[attack/ai/rogue_actions]]
- [[attack/ai/insecure_ai_components]]
- [[attack/ai/vulnerable_ai_systems]]
- [[attack/ai/denial_of_ml_service]]
- [[attack/ai/model_reverse_engineering]]
- [[attack/ai/mcp_security]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/adversarial_examples]]
- [[attack/ai/jailbreaking]]
- [[attack/ai/_overview]]
- [[definitions/owasp_llm_top10]]

## Sources
- raw/attacking_ai-application_and_systems/overview_of_application&_system_components.md
- raw/attacking_ai-application_and_systems/attacking_the_application_denial_of_ml_service.md
- raw/attacking_ai-application_and_systems/attacking_the_application_insecure_integrated_components.md
- raw/attacking_ai-application_and_systems/attacking_the_applications_model_reverse_engineering.md
- raw/attacking_ai-application_and_systems/attacking_the_applications_rogue_actions.md
- raw/attacking_ai-application_and_systems/attacking_the_system_excessive_data_handling_&_insecure_storage.md
- raw/attacking_ai-application_and_systems/attacking_the_system_model_deployment_tampering.md
- raw/attacking_ai-application_and_systems/attacking_the_system_vulnerable_framework_code.md
