---
tags: [definition, concept, reference]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 1
---

# CVSS — Common Vulnerability Scoring System

An open industry standard for calculating the severity of a vulnerability on a 0–10 scale. CVSS v3.1 is the current version; v4.0 is emerging. Used by NVD, Nessus, OpenVAS, and almost every scanner to triage findings.

## Overview

CVSS produces three independent scores, each reflecting a different context:

| Group | What it measures |
|-------|----------------|
| **Base** | Intrinsic characteristics of the vulnerability (unchanged regardless of environment) |
| **Temporal** | How exploitable the issue is *right now* (exploit availability, patch status) |
| **Environmental** | How significant it is *in your specific environment* (modified CIA values) |

The Base Score is the one most commonly cited. Temporal and Environmental scores are optional refinements.

## Base Metric Group

### Exploitability Metrics

These measure *how hard it is for an attacker to exploit the vulnerability*.

| Metric | Options | Notes |
|--------|---------|-------|
| **Attack Vector (AV)** | Network (N) / Adjacent (A) / Local (L) / Physical (P) | N = remotely exploitable from internet; highest severity |
| **Attack Complexity (AC)** | Low (L) / High (H) | Low = no special conditions required |
| **Privileges Required (PR)** | None (N) / Low (L) / High (H) | None = unauthenticated; highest severity |
| **User Interaction (UI)** | None (N) / Required (R) | None = no victim action needed |

### Scope (S)

Whether exploitation can affect components beyond the vulnerable component itself.
- **Unchanged (U):** Impact stays within the vulnerable component
- **Changed (C):** Exploitation can impact other components (e.g., hypervisor escape → host OS)

### Impact Metrics (CIA Triad)

| Metric | None | Low | High |
|--------|------|-----|------|
| **Confidentiality (C)** | No data exposed | Partial | Full data disclosure |
| **Integrity (I)** | No modification | Partial | Full modification possible |
| **Availability (A)** | No disruption | Partial | Full denial of service |

### Base Score Severity Bands

| Score | Severity |
|-------|----------|
| 0.0 | None |
| 0.1–3.9 | Low |
| 4.0–6.9 | Medium |
| 7.0–8.9 | High |
| 9.0–10.0 | Critical |

**Priority target formula:** CVSS ≥ 9.0 with `AV:N` (network-accessible) and `PR:N` (no privileges required) → act immediately.

## Temporal Metric Group

Temporal metrics adjust the Base Score based on current real-world conditions. Scores drift over time as exploits are published and patches released.

| Metric | Options |
|--------|---------|
| **Exploit Code Maturity (E)** | Not Defined / Unproven / Proof-of-Concept / Functional / High |
| **Remediation Level (RL)** | Not Defined / Official Fix / Temporary Fix / Workaround / Unavailable |
| **Report Confidence (RC)** | Not Defined / Unknown / Reasonable / Confirmed |

**Pentester note:** A CVE with `E:F` (Functional exploit publicly available) and `RL:U` (no patch) is higher priority than the raw Base Score alone suggests — temporal score can be decisive in triage.

## Environmental Metric Group

Environmental metrics let defenders modify the Base Score to reflect their actual exposure. Rarely filled in during pentests but important for client-side risk management.

- **Modified Base Metrics:** Override any Base metric for the specific environment (e.g., if a component is network-isolated, set `AV:L` regardless of the published `AV:N`).
- **Confidentiality/Integrity/Availability Requirements (CR/IR/AR):** Set to Low/Medium/High to weight CIA impact by business criticality.

## DREAD Risk Assessment (Microsoft)

DREAD is a complementary scoring system often used alongside CVSS, especially in application security. It uses a 1–10 scale across five factors, averaged to produce an overall risk score.

| Factor | Question |
|--------|----------|
| **Damage Potential** | How much damage if exploited? |
| **Reproducibility** | How easily can the exploit be reproduced? |
| **Exploitability** | How easy is it to launch an attack? |
| **Affected Users** | How many users are affected? |
| **Discoverability** | How easy is it to discover the vulnerability? |

DREAD is more subjective than CVSS and has largely been superseded by CVSS v3 for formal reporting, but it appears in legacy reports and some application-focused methodologies.

## Calculating CVSS Scores

The NVD CVSS calculator is at `https://nvd.nist.gov/vuln-metrics/cvss`. Most scanners (Nessus, OpenVAS) calculate and display CVSS automatically from plugin/NVT data.

```bash
# nvdcli — unofficial CVSS lookup from the terminal
pip install nvdlib
python3 -c "import nvdlib; r = nvdlib.searchCVE(cveId='CVE-2021-34527')[0]; print(r.score)"
```

## Gotchas & Notes

- CVSS measures technical severity, not business risk. A CVSS 9.8 on an isolated dev server may be lower priority than a CVSS 6.5 on an internet-facing payment system.
- Environmental scores are almost never filled in by scanner vendors — the published Base Score assumes worst case.
- CVSS 4.0 (released 2023) adds four metric groups (Base, Threat, Environmental, Supplemental) and renames Temporal to Threat. NVD is still primarily publishing v3.1 scores.
- Zero-days have no CVE and no CVSS — document them with a manual severity assessment.

## Related Pages

- [[definitions/cve_and_oval]] — the CVE identifier system that pairs with CVSS scores
- [[definitions/security_terminology]] — brief CVSS summary alongside other key terms
- [[definitions/vulnerability_assessment]] — VA methodology that produces CVSS-scored findings
- [[definitions/va_reporting]] — how to present CVSS scores in client reports
- [[tools/enumeration/nessus]] — Nessus plugin severity levels map to CVSS
- [[tools/enumeration/openvas]] — OpenVAS NVT severity ratings

## Sources

- raw/vulnerability_assessment/common_vulnerability_scoring_system.md
