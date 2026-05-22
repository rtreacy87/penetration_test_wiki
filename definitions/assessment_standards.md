---
tags: [definition, concept, reference]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 1
---

# Assessment Standards

Compliance and methodology standards that govern how vulnerability assessments and penetration tests must be conducted to satisfy legal, regulatory, or industry requirements.

## Overview

Standards matter for two reasons: (1) regulators or clients require them by contract, and (2) they define minimum rigor so that findings are defensible. Compliance standards dictate *what* must be tested; pentest methodology standards dictate *how* to test it.

## Compliance Standards

### PCI DSS (Payment Card Industry Data Security Standard)

Governs any organization that stores, processes, or transmits cardholder data. Not a government regulation but contractually required by card brands.

**Scanning requirements:**
- Internal and external vulnerability scans required quarterly
- Scans must be performed by a PCI-approved scanning vendor (ASV) for external scans
- Cardholder data must reside in a segmented Cardholder Data Environment (CDE)

**Pentester relevance:** Scope is often limited to the CDE. Verify network segmentation is actually enforced — if cardholder data leaks outside the CDE boundary, scope expands.

---

### HIPAA (Health Insurance Portability and Accountability Act)

Protects patient health information (PHI). A U.S. federal law.

**Scanning requirements:**
- No mandatory VA schedule, but a risk assessment and vulnerability identification are required
- The Security Rule's risk analysis requirement is typically satisfied by documented VA results

**Pentester relevance:** PHI systems must be handled carefully — avoid destroying logs or disrupting clinical systems. Scope clauses almost always exclude life-safety systems.

---

### FISMA (Federal Information Security Management Act)

Governs U.S. federal agencies and their contractors. Implements NIST SP 800-53 controls.

**Scanning requirements:**
- Requires a documented vulnerability management program
- Continuous monitoring (ISCM) mandated — not just point-in-time scans
- Documentation and proof of program activity required for audits

**Pentester relevance:** Government contracts often require FISMA compliance; testing must produce artifacts that satisfy ISCM documentation requirements.

---

### ISO 27001

International standard for information security management systems (ISMS). Certification is awarded by accredited third-party auditors.

**Scanning requirements:**
- Quarterly internal and external vulnerability scans required
- Must be part of a broader ISMS with defined policies, risk treatment, and continual improvement

**Pentester relevance:** ISO 27001 audits evaluate the *process* around security, not just technical controls. Pentest findings must feed into the organization's risk register and be tracked to remediation.

---

## Penetration Testing Methodology Standards

These define the phases and scope of a pentest engagement. Use them to structure proposals, scope documents, and reports.

### PTES (Penetration Testing Execution Standard)

A comprehensive framework applicable to all pentest types.

| Phase | Focus |
|-------|-------|
| Pre-engagement Interactions | Rules of engagement, scope, legal agreements |
| Intelligence Gathering | Passive and active recon |
| Threat Modeling | Identify assets and attack vectors |
| Vulnerability Analysis | Identify and validate vulnerabilities |
| Exploitation | Attempt to compromise targets |
| Post Exploitation | Measure impact, lateral movement, persistence |
| Reporting | Document findings and remediation steps |

---

### OSSTMM (Open Source Security Testing Methodology Manual)

Divided into five operational security channels, each with defined metrics and scope controls.

| Channel | Scope |
|---------|-------|
| Human Security | Social engineering, phishing |
| Physical Security | Physical access, locks, badges |
| Wireless Communications | Wi-Fi, Bluetooth, RFID |
| Telecommunications | VoIP, PSTN |
| Data Networks | Network and application testing |

**OSSTMM is unique** in that it produces a *security metric* (RVSS — Rule of Least Privilege Verification Score) rather than just a list of findings. It quantifies actual attack surface.

---

### NIST Penetration Testing Framework

From NIST SP 800-115 (Technical Guide to Information Security Testing and Assessment).

| Phase | Activity |
|-------|----------|
| Planning | Define objectives, scope, rules of engagement |
| Discovery | Reconnaissance and vulnerability identification |
| Attack | Exploitation of identified vulnerabilities |
| Reporting | Document findings, risk, and remediation |

---

### OWASP Testing Frameworks

OWASP maintains several testing guides depending on the target:

| Guide | Target |
|-------|--------|
| WSTG (Web Security Testing Guide) | Web applications |
| MSTG (Mobile Security Testing Guide) | Mobile applications (iOS/Android) |
| FSTM (Firmware Security Testing Methodology) | Embedded devices and IoT firmware |

**OWASP Top 10** (see [[definitions/security_terminology]]) is the most commonly cited output artifact — clients often request findings mapped to OWASP Top 10 categories.

---

## Gotchas & Notes

- Compliance-driven scanning is a floor, not a ceiling. PCI DSS quarterly scans do not replace a risk-based vulnerability management program.
- Signed rules of engagement (RoE) and scope documents are legally required before any pentest begins — these are not optional regardless of which standard applies.
- PTES and OSSTMM are not mutually exclusive — many engagements blend phases from both.
- When scoping for compliance, identify which standards apply *before* agreeing to scope — HIPAA clinical systems exclusions and FISMA documentation requirements both affect what you can do and what you must deliver.

## Related Pages

- [[definitions/vulnerability_assessment]] — VA methodology and types of assessments
- [[definitions/cvss]] — CVSS scoring used in all compliance reporting
- [[definitions/cve_and_oval]] — CVE and OVAL as compliance scan inputs
- [[definitions/va_reporting]] — how to structure deliverables for compliance engagements

## Sources

- raw/vulnerability_assessment/assessment_standards.md
