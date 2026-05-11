# Wiki Log

## [2026-05-11] structure | Added definitions/, shell_commands/, ports/ sections
Pages created: 10 (definitions/_overview, network_protocols, tcp_flags, auth_protocols, security_terminology; ports/common_ports; shell_commands/bash, cmd, powershell). Pages updated: 3 (CLAUDE.md schema, index.md, log.md).
Key additions: TCP/UDP/ICMP/SCTP/ARP/QUIC reference with pentester context; TCP flag → scan-type inference table; NTLM/Kerberos/OAuth/SAML/JWT attack paths; CVE/CVSS/RCE/LFI/SSRF/XSS/SSTI terminology; 80+ port quick-reference table with pentester interest column; bash/cmd/PowerShell command sheets covering enum, file transfer, cred hunting, reverse shells, lateral movement.

## [2026-05-11] ingest | HTB Academy — Attacking Common Services (raw/attacking_common_services/)
Files read: 19. Pages created: 10 (attack/ftp, attack/smb, attack/dns, attack/smtp, attack/rdp, attack/sql_databases, tools/responder, tools/medusa, tools/hydra, tools/crowbar). Pages updated: 6 (attack/_overview, enumeration/smb, enumeration/ftp, enumeration/dns, enumeration/smtp, index.md).
Key additions: FTP bounce + CVE-2022-22836, SMB Responder/NTLM relay + CVE-2020-0796 SMBGhost, DNS subdomain takeover + cache poisoning, SMTP O365spray + CVE-2020-7247 OpenSMTPD RCE, RDP tscon session hijacking + BlueKeep CVE-2019-0708, MSSQL xp_cmdshell/xp_dirtree/IMPERSONATE/linked server lateral movement.

## [2026-05-10] ingest | HTB Academy — AI Data Attacks (raw/ai_data_attacks/)
Files read: 25. Pages created: 5. Pages updated: 3 (_overview, index, CLAUDE.md).
Key additions: data poisoning hub (pipeline stage mapping, OWASP LLM03/LLM05), label flipping + targeted label attacks (decision boundary math, implementation), clean label attacks (feature-perturbation without label changes), trojan/backdoor attacks (GTSRB CNN, 100% ASR at 10% poison rate), model steganography (pickle __reduce__ RCE + IEEE 754 LSB encoding, reverse shell chain).

## [2026-05-10] ingest | Full wiki build — 6 modules ingested simultaneously

Files read: 106. Pages created: 57. Pages updated: 0.

**Modules ingested:**

- `raw/footprinting/` (21 files) — 28 pages created: 2 methodology overviews, 3 OSINT/recon pages, 13 service enumeration pages (DNS, FTP, SMTP, SMB, SNMP, NFS, IMAP/POP3, IPMI, MSSQL, MySQL, Oracle TNS, Linux remote mgmt, Windows remote mgmt), 10 tool pages (smbclient, enum4linux, rpcclient, snmpwalk, onesixtyone, dnsenum, dig, crackmapexec, impacket, odat), 3 HTB lab write-ups (easy/medium/hard).
- `raw/network_enumeration_with_nmap/` (13 files) — 2 pages created: comprehensive nmap tool reference and firewall evasion attack page.
- `raw/linux_privilege_escalation/` (26 files) — 7 pages created: hub page + 4 sub-pages covering enumeration, sudo/SUID/capabilities, services (Docker/LXD/K8s/cron), kernel exploits and library hijacking; plus linpeas and pspy tool pages.
- `raw/prompt_injection_attacks/` (12 files) — 4 pages created: AI attack surface overview, direct/indirect prompt injection, jailbreaking techniques (7 families), and defense-in-depth mitigations analysis.
- `raw/ai_evasion_sparsity/` (28 files) — 3 pages created: adversarial ML threat model overview, JSMA (single-pixel and pairwise), ElasticNet/EAD attack with FISTA implementation.
- `raw/attacking_ai-application_and_systems/` (13 files) — 4 pages created: attacking AI systems hub, DoML/sponge examples, model reverse engineering/extraction, MCP protocol security (architecture + attack surface + mitigations).

**Scaffolding pages created (7):** `index.md`, `log.md`, `tools/_overview.md`, `attack/_overview.md`, `labs/_overview.md`, `wordlists/_overview.md`, `wordlists/use_cases.md`.

**Key additions:**
- Full service enumeration library (13 services) from HTB Academy Footprinting module
- Comprehensive nmap reference covering all scan types, NSE categories, performance tuning, and IDS evasion
- Linux privilege escalation complete reference: environment abuses, permission-based techniques, service escapes (Docker/LXD/K8s), kernel CVEs (Dirty Pipe, PwnKit, Baron Samedit), library hijacking
- AI/LLM attack coverage: prompt injection taxonomy, jailbreak technique families with concrete examples, adversarial ML (JSMA + EAD), model extraction, DoML, MCP security
- 3 HTB Footprinting lab write-ups documenting multi-service attack chains
