---
tags: [definition, concept, reference]
module: core
last_updated: 2026-05-21
source_count: 0
---

# Definitions Overview

Pentest-oriented reference definitions — protocols, flags, authentication mechanisms, and security terminology.

## Pages

| Page | Covers |
|------|--------|
| [[definitions/network_protocols]] | TCP, UDP, ICMP, SCTP, ARP, QUIC, GRE/IPSec — what they are and pentester relevance |
| [[definitions/tcp_flags]] | SYN, ACK, FIN, RST, PSH, URG — flag behavior, three-way handshake, scan type inference |
| [[definitions/auth_protocols]] | NTLM, Kerberos, LDAP, Basic, Digest, OAuth, SAML, JWT — how they fail |
| [[definitions/security_terminology]] | CVE, CVSS, RCE, LFI, SSRF, IDOR, SQLi, XSS, XXE, SSTI, lateral movement, persistence |
| [[definitions/dns]] | DNS primer: zones, records, zone transfers, AXFR, detecting restricted vs unrestricted transfers |
| [[definitions/owasp_llm_top10]] | OWASP LLM Top 10 (2025): LLM01–LLM10 with attack technique mapping and wiki page cross-references |
| [[definitions/vulnerability_assessment]] | VA concept: 8-step methodology, key terms (vuln/threat/exploit/risk), asset management, VA vs pentest types |
| [[definitions/assessment_standards]] | Compliance standards (PCI DSS, HIPAA, FISMA, ISO 27001) and pentest frameworks (PTES, OSSTMM, NIST, OWASP) |
| [[definitions/cvss]] | CVSS v3.1 deep dive: base/temporal/environmental metric groups, DREAD, severity bands, priority formula |
| [[definitions/cve_and_oval]] | CVE catalog, OVAL XML standard, stages of CVE disclosure, responsible disclosure, SCAP integration |
| [[definitions/va_reporting]] | VA/pentest report structure: executive summary, scope, per-finding fields, Nessus/OpenVAS export commands |

## How definitions pages work

Each page explains *what something is* in terms useful to a practitioner — not a textbook definition, but the version that tells you why you care during an engagement. Every definition links to the enumeration or attack page that exploits it.

## Related pages

- [[ports/common_ports]] — port → service → pentester interest
- [[shell_commands/bash]] — Linux quick-reference
- [[shell_commands/cmd]] — Windows CMD quick-reference
- [[shell_commands/powershell]] — PowerShell quick-reference
- [[enumeration/_overview]]
- [[attack/_overview]]
