---
tags: [attack, concept]
module: multi
last_updated: 2026-05-11
source_count: 6
---

# Attack Overview

High-level map of the exploitation and post-exploitation techniques documented in this wiki.

## Overview

Attacks are organized by target: network infrastructure, Linux hosts, web applications, and AI/ML systems. Each category has sub-pages with specific techniques.

## Attack categories

### Network attacks
Techniques targeting network services and firewalls during enumeration and exploitation.

- [[attack/network/firewall_evasion]] — Nmap-based IDS/firewall evasion: fragmentation, decoys, source port manipulation

### Linux privilege escalation
Post-exploitation techniques for escalating from low-privilege shell to root on Linux.

- [[attack/linux_privilege_escalation]] — Hub page: all LPE categories with quick-wins checklist
  - [[attack/linux_privesc_enumeration]] — What to look for and how after initial shell
  - [[attack/linux_privesc_sudo_suid]] — Sudo abuse, SUID/SGID, capabilities, privileged groups
  - [[attack/linux_privesc_services]] — Cron abuse, Docker/LXD escape, Kubernetes, logrotate
  - [[attack/linux_privesc_kernel]] — Kernel CVEs, library hijacking, recent 0-days

### AI / LLM attacks
Techniques targeting AI systems, LLMs, and ML models.

- [[attack/ai/_overview]] — Full map of the AI attack surface
  - [[attack/ai/prompt_injection]] — Direct and indirect prompt injection
  - [[attack/ai/jailbreaking]] — LLM jailbreak technique families
  - [[attack/ai/prompt_injection_mitigations]] — Defenses and their effectiveness
  - [[attack/ai/adversarial_examples]] — Threat model for ML evasion attacks
  - [[attack/ai/jsma]] — Jacobian-Based Saliency Map Attack
  - [[attack/ai/elasticnet_attack]] — EAD: sparse adversarial examples via FISTA
  - [[attack/ai/attacking_ai_systems]] — Full AI system attack surface (app + system layers)
  - [[attack/ai/denial_of_ml_service]] — Sponge examples and DoML techniques
  - [[attack/ai/model_reverse_engineering]] — Model extraction / stealing attacks
  - [[attack/ai/mcp_security]] — MCP protocol attack surface and mitigations

### Common services attacks
Exploitation techniques for FTP, SMB, DNS, SMTP, RDP, and SQL — protocol-specific attacks after initial enumeration.

- [[attack/ftp]] — Anonymous login, Medusa brute force, FTP bounce, CVE-2022-22836 path traversal RCE
- [[attack/smb]] — Null sessions, CME password spray, psexec/smbexec RCE, SAM dump, PTH, Responder LLMNR, NTLM relay, CVE-2020-0796
- [[attack/dns]] — AXFR zone transfer, subdomain enum, subdomain takeover (CNAME), DNS cache poisoning
- [[attack/smtp]] — VRFY/RCPT user enum, O365spray, Hydra spraying, open relay + swaks, CVE-2020-7247 OpenSMTPD RCE
- [[attack/rdp]] — Crowbar/Hydra spraying, session hijacking (tscon), Pass-the-Hash (xfreerdp), CVE-2019-0708 BlueKeep
- [[attack/sql_databases]] — xp_cmdshell RCE, file read/write, xp_dirtree hash stealing, IMPERSONATE privesc, linked server lateral movement

### Web attacks
_(pages to be added)_

Common web attack pages to create next: `attack/web/sqli.md`, `attack/web/xss.md`, `attack/web/lfi_rfi.md`, `attack/web/directory_traversal.md`.

## Attack flow (general engagement)

```
Recon (passive) → [[recon/_overview]]
  ↓
Enumeration (active) → [[enumeration/_overview]]
  ↓
Exploitation
  ├─ Network service exploits → [[attack/network/]]
  ├─ Linux host access → initial shell
  │     └─ Privilege escalation → [[attack/linux_privilege_escalation]]
  ├─ Web application → [[attack/web/]] (TBD)
  └─ AI/ML system → [[attack/ai/_overview]]
```

## Related pages

- [[enumeration/_overview]]
- [[recon/_overview]]
- [[tools/_overview]]
