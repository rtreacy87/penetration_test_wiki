# Wiki Index

_Last updated: 2026-05-12 — 123 pages total_

---

## Recon

- [[recon/_overview]] — Passive vs active recon, six-layer model, recommended order of operations. `recon` `concept`
- [[recon/osint/domain_information]] — WHOIS, certificate transparency (crt.sh), DNS history, SecurityTrails, Shodan. `recon` `recon/osint`
- [[recon/osint/cloud_resources]] — AWS S3, Azure blob, GCP storage discovery via DNS, Google dorks, GrayHatWarfare. `recon` `recon/osint`
- [[recon/osint/staff_enumeration]] — LinkedIn employee discovery, job posting fingerprinting, GitHub credential exposure, email patterns. `recon` `recon/osint`

---

## Enumeration

- [[enumeration/_overview]] — What enumeration is, the six-layer model, infrastructure vs host-based, the enumeration loop. `enumeration` `concept`
- [[enumeration/dns]] — DNS record types, zone transfers (AXFR), subdomain brute-forcing, dig/dnsenum/host syntax. `enumeration` `enumeration/dns`
- [[enumeration/ftp]] — Anonymous login, vsFTPd config, nmap NSE scripts, recursive listing, TLS cert disclosure. `enumeration` `enumeration/ftp`
- [[enumeration/smtp]] — VRFY/EXPN user enumeration, open relay detection, nmap scripts, telnet interaction. `enumeration` `enumeration/smtp`
- [[enumeration/smb]] — SMB versions, null sessions, rpcclient RID brute-force, smbclient, enum4linux, NetExec (nxc). `enumeration` `enumeration/smb`
- [[enumeration/snmp]] — SNMP versions, community strings, MIB/OID structure, snmpwalk, onesixtyone, braa. `enumeration` `enumeration/snmp`
- [[enumeration/nfs]] — NFS exports, showmount, UID/GID-based access, SUID escalation via mounted share. `enumeration` `enumeration/nfs`
- [[enumeration/imap_pop3]] — IMAP vs POP3, openssl TLS sessions, curl access, command reference. `enumeration` `enumeration/imap_pop3`
- [[enumeration/ipmi]] — BMC implementations, RAKP IPMI 2.0 hash extraction, Metasploit modules, Hashcat mode 7300. `enumeration` `enumeration/ipmi`
- [[enumeration/mssql]] — Default databases, Windows vs SQL auth, xp_cmdshell, linked servers, impacket-mssqlclient. `enumeration` `enumeration/mssql`
- [[enumeration/mysql]] — information_schema, dangerous config settings, nmap scripts, LOAD_FILE file read. `enumeration` `enumeration/mysql`
- [[enumeration/oracle_tns]] — TNS config, SID brute-forcing, ODAT modules, sqlplus sysdba escalation, file upload. `enumeration` `enumeration/oracle_tns`
- [[enumeration/linux_remote_mgmt]] — SSH auth methods/dangerous settings/ssh-audit, Rsync daemon enum, R-services. `enumeration` `protocol`
- [[enumeration/windows_remote_mgmt]] — RDP (NLA, xfreerdp, rdp-sec-check), WinRM (evil-winrm), WMI (wmiexec). `enumeration` `protocol`

---

## Attack

- [[attack/_overview]] — Map of all attack categories: network, common services (FTP/SMB/DNS/SMTP/RDP/SQL), Linux privesc, web, AI/ML. `attack` `concept`

### Linux Privilege Escalation

- [[attack/linux_privilege_escalation]] — Hub page: all LPE categories, quick-wins checklist, methodology flow. `attack` `attack/privesc`
- [[attack/linux_privesc_enumeration]] — Post-exploitation enumeration: OS, users, network, files, services, creds, procfs. `attack` `attack/privesc`
- [[attack/linux_privesc_sudo_suid]] — Sudo abuse, SUID/SGID exploitation, capabilities, privileged group abuse (docker, lxd). `attack` `attack/privesc`
- [[attack/linux_privesc_services]] — Cron job abuse, Docker escape, LXD, Kubernetes, logrotate, Screen, NFS, tmux. `attack` `attack/privesc`
- [[attack/linux_privesc_kernel]] — Dirty Pipe, Netfilter CVEs, PwnKit/Polkit, Baron Samedit, library hijacking, Python hijacking. `attack` `attack/privesc`

### Network Attacks

- [[attack/network/firewall_evasion]] — Nmap-based firewall/IDS evasion: fragmentation, decoys, source port, timing. `attack` `attack/network`

### Common Services Attacks

- [[attack/ftp]] — Anonymous login, Medusa brute force, FTP bounce scan, CVE-2022-22836 CoreFTP path traversal. `attack` `attack/network`
- [[attack/smb]] — Null sessions, CME spray, psexec/smbexec RCE, SAM dump, PTH, Responder LLMNR, NTLM relay, CVE-2020-0796. `attack` `attack/network`
- [[attack/dns]] — AXFR exploitation, subdomain enum (subfinder/subbrute), CNAME subdomain takeover, DNS cache poisoning. `attack` `attack/network`
- [[attack/smtp]] — VRFY/RCPT user enum, O365spray, Hydra spraying, swaks open relay abuse, CVE-2020-7247 OpenSMTPD RCE. `attack` `attack/network`
- [[attack/rdp]] — Crowbar/Hydra password spray, tscon session hijacking, xfreerdp PTH, CVE-2019-0708 BlueKeep. `attack` `attack/network`
- [[attack/sql_databases]] — xp_cmdshell RCE, OUTFILE webshell, xp_dirtree NTLMv2 theft, IMPERSONATE privesc, linked server pivot. `attack` `attack/network`

### AI / LLM Attacks

- [[attack/ai/_overview]] — Full map of AI attack surface: injection, jailbreaking, adversarial ML, data poisoning, MCP, system attacks. `attack` `attack/ai` `concept`
- [[attack/ai/prompt_injection]] — Direct and indirect prompt injection: 6 direct strategies, 3 indirect scenarios, payload examples. `attack` `attack/ai`
- [[attack/ai/jailbreaking]] — 7 jailbreak technique families: DAN, roleplay, token smuggling, adversarial suffix, IMM. `attack` `attack/ai`
- [[attack/ai/prompt_injection_mitigations]] — Defenses across 7 control layers: prompt engineering through guardrail LLMs. `attack` `attack/ai` `concept`
- [[attack/ai/adversarial_examples]] — Threat model for ML evasion: white-box vs black-box, L0/L1/L2/Linf norms. `attack` `attack/ai` `concept`
- [[attack/ai/jsma]] — JSMA: Jacobian saliency maps, single-pixel vs pairwise attacks, implementation, tradeoffs. `attack` `attack/ai`
- [[attack/ai/elasticnet_attack]] — EAD: L1+L2 sparse adversarial examples via FISTA, binary search, C&W comparison. `attack` `attack/ai`
- [[attack/ai/attacking_ai_systems]] — Hub: application attacks (DoML, insecure components, model RE, rogue actions) + system attacks. `attack` `attack/ai`
- [[attack/ai/denial_of_ml_service]] — DoML: sponge examples, resource exhaustion via adversarial inputs, genetic algorithm. `attack` `attack/ai`
- [[attack/ai/model_reverse_engineering]] — Model extraction: query-based stealing, membership inference, surrogate training. `attack` `attack/ai`
- [[attack/ai/mcp_security]] — MCP protocol: architecture, malicious servers, vulnerable servers, tool poisoning, mitigations. `attack` `attack/ai` `protocol`
- [[attack/ai/llm_reconnaissance]] — LLM recon: model identity probing, architecture mapping (RAG/plugins/agents), safeguard detection, llmmap fingerprinting, attack surface map template. `attack` `attack/ai` `recon`
- [[attack/ai/rogue_actions]] — Excessive agency exploitation: direct plugin bypass ("I am an administrator"), indirect injection via stored user data → privileged admin context. `attack` `attack/ai`
- [[attack/ai/insecure_ai_components]] — Web/plugin vulns in AI apps: IDOR on LLM history endpoints, SQLi, plugin auth bypass via prompt injection, output injection. `attack` `attack/ai`
- [[attack/ai/vulnerable_ai_systems]] — System-layer attacks: exposed .db files (LLM logs with PII), ShellTorch RCE chain (CVE-2023-43654 + CVE-2022-1471), MLflow LFI (CVE-2023-6909, CVE-2024-1594), Ollama DoS (CVE-2025-1975). `attack` `attack/ai`
- [[attack/ai/data_poisoning]] — Hub: AI pipeline attack surface, OWASP LLM03/LLM05 mapping, all data attack types. `attack` `attack/ai` `concept`
- [[attack/ai/label_flipping]] — Label flipping and targeted label attacks: decision boundary math, flip_labels implementation, evaluation. `attack` `attack/ai`
- [[attack/ai/clean_label_attacks]] — Feature-perturbation poisoning: target selection, perturbation vector, one-step boundary shift without touching labels. `attack` `attack/ai`
- [[attack/ai/trojan_attacks]] — Backdoor attacks: trigger embedding, CNN architecture, Clean Accuracy vs ASR (100% on GTSRB Stop→Speed 60). `attack` `attack/ai`
- [[attack/ai/model_steganography]] — Pickle `__reduce__` RCE + tensor LSB steganography: full reverse shell chain via malicious `.pth` file. `attack` `attack/ai`

---

## Tools

- [[tools/_overview]] — Tool selection guide: comparison tables by function and decision tree by scenario. `tool` `concept`

### Utility

- [[tools/utility/ssh]] — SSH client: password/key login, key pair creation/deletion, known_hosts fingerprint management for lab VMs. `tool`
- [[tools/utility/xfreerdp]] — Modern RDP client; NLA, Pass-the-Hash (`/pth`), RDP 7–10, drive sharing, clipboard. Preferred for all engagements. `tool`
- [[tools/utility/rdesktop]] — Legacy RDP client; RDP 5 only, no NLA, no PTH; use only for Windows XP/Server 2003 targets. `tool`
- [[tools/utility/sqlcmd]] — Microsoft SQL Server CLI; batch mode (`GO`), non-interactive `-Q`, install via Microsoft repo on Linux. `tool`
- [[tools/utility/impacket]] — Suite: mssqlclient, psexec, secretsdump, wmiexec, GetUserSPNs, samrdump, smbserver. `tool`
- [[tools/utility/crackmapexec]] — LEGACY: predecessor to NetExec; identical syntax, no longer maintained; includes removal instructions. `tool`
- [[tools/utility/ollama]] — Local LLM runtime: model management, VRAM sizing, model comparison by pentesting role, PyRIT integration as complete attack harness. `tool` `attack/ai`

### Enumeration

- [[tools/enumeration/nmap]] — Port scanning, service detection, NSE scripts, OS detection, performance, firewall evasion. `tool`
- [[tools/enumeration/smbclient]] — SMB share browsing, file operations, null session syntax. `tool`
- [[tools/enumeration/enum4linux]] — enum4linux-ng: SMB/NetBIOS user/group/share enumeration. `tool`
- [[tools/enumeration/rpcclient]] — RPC enumeration commands, RID brute-force, queryuser/enumdomusers. `tool`
- [[tools/enumeration/snmpwalk]] — MIB tree walking, key OID subtrees, community string usage. `tool`
- [[tools/enumeration/onesixtyone]] — SNMP community string brute-force; use before snmpwalk. `tool`
- [[tools/enumeration/dig]] — DNS query tool: all record types, AXFR syntax, output format, bash brute-force loop. `tool`
- [[tools/enumeration/dnsenum]] — DNS enumeration: NS/MX/AXFR/brute-force in one run. `tool`
- [[tools/enumeration/subbrute]] — DNS subdomain brute-forcer; use `-r resolvers.txt` to target a specific nameserver for internal/HTB domains. `tool`
- [[tools/enumeration/smtp_user_enum]] — SMTP user enumeration via VRFY/EXPN/RCPT TO; RCPT mode works even when VRFY is disabled. `tool`
- [[tools/enumeration/garak]] — Automated LLM vulnerability scanner: DAN/promptinject probe families, resilience scoring, JSON+HTML reports. `tool` `attack/ai`
- [[tools/enumeration/llmmap]] — LLM fingerprinting via 8-probe interactive session; identifies model family without API access. `tool` `attack/ai`
- [[tools/enumeration/linpeas]] — Automated Linux privilege escalation enumeration; staged to /opt, run on target. `tool`
- [[tools/enumeration/pspy]] — Unprivileged process monitor; catches cron and credential leaks; run on target. `tool`

### Attack

- [[tools/attack/netexec]] — NetExec (nxc): credential spray, validation, PTH, and enumeration across SMB/WinRM/MSSQL/LDAP/SSH. `tool`
- [[tools/attack/metasploit]] — msfconsole: module framework for exploitation, smb_login/ssh_login/ftp_login brute-force, Meterpreter. `tool`
- [[tools/attack/responder]] — LLMNR/NBT-NS poisoner; captures NTLMv2 hashes from Windows broadcast auth. `tool`
- [[tools/attack/medusa]] — Parallel brute-forcer; more reliable than Hydra for FTP/SSH. `tool`
- [[tools/attack/hydra]] — Versatile network login brute-forcer; 50+ protocols, go-to for HTTP/RDP/SMTP/POP3. `tool`
- [[tools/attack/crowbar]] — RDP-focused brute-forcer; more reliable than Hydra for RDP password spraying. `tool`
- [[tools/attack/odat]] — Oracle attack tool: SID enum, auth brute-force, web shell upload. `tool`
- [[tools/attack/pyrit]] — Microsoft AI Red Team framework: orchestrates LLM attacks (jailbreaking, prompt injection), Ollama local + Claude cloud setup, RTX 4090 model guide. `tool` `attack/ai`
- [[tools/attack/art]] — Adversarial Robustness Toolbox: JSMA, EAD, FGSM, PGD, C&W, poisoning attacks; PyTorch/TF/sklearn wrappers; white-box and black-box modes. `tool` `attack/ai`

---

## Definitions

- [[definitions/_overview]] — Index of all definition pages: protocols, flags, auth, security terms. `definition` `concept`
- [[definitions/network_protocols]] — TCP, UDP, ICMP, SCTP, ARP, QUIC — behavior and pentester relevance for each. `definition` `concept`
- [[definitions/tcp_flags]] — SYN/ACK/FIN/RST/PSH/URG — three-way handshake, port state inference, scan type selection. `definition` `reference`
- [[definitions/auth_protocols]] — NTLM, Kerberos, LDAP, OAuth, SAML, JWT — how they work and how they fail. `definition` `concept`
- [[definitions/security_terminology]] — CVE/CVSS, RCE, LFI, SSRF, IDOR, SQLi, XSS, SSTI, lateral movement, persistence, defense terms. `definition` `concept`
- [[definitions/dns]] — DNS primer: records (A/MX/TXT/NS/CNAME), zones, zone transfers, AXFR explained, how to detect restricted vs unrestricted transfers. `definition` `concept`
- [[definitions/owasp_llm_top10]] — OWASP LLM Top 10 (2025): LLM01–LLM10 with attack technique mapping, exam-day quick-reference table, and links to every relevant wiki page. `definition` `reference` `attack/ai`

---

## Ports

- [[ports/common_ports]] — Master port reference: remote access, file transfer, web, email, databases, directory services, infrastructure. `reference` `enumeration`

---

## Shell Commands

- [[shell_commands/bash]] — Linux bash: system enum, file search, creds hunting, reverse shells, file transfer, listener. `shell` `reference`
- [[shell_commands/cmd]] — Windows CMD: whoami/net/wmic, registry, file transfer (certutil/bitsadmin), RDP session hijack. `shell` `reference`
- [[shell_commands/powershell]] — PowerShell: AD enum, file transfer, WinRM remoting, AMSI bypass, reverse shell. `shell` `reference`

---

## Wordlists

- [[wordlists/_overview]] — SecLists structure, custom generation (CeWL, crunch, hashcat rules, username-anarchy). `wordlist` `concept`
- [[wordlists/use_cases]] — Task-to-wordlist mapping: web dirs, DNS, SNMP, usernames, passwords; tool-specific examples. `wordlist`

---

## Labs

- [[labs/_overview]] — Lab write-up index by platform and difficulty. `lab` `concept`

### Footprinting

- [[labs/htb/footprinting_easy]] — DNS zone transfer + NFS SSH key extraction → flag. `lab`
- [[labs/htb/footprinting_medium]] — Multi-service (FTP/SMB/NFS/SNMP) chaining to obtain credentials. `lab`
- [[labs/htb/footprinting_hard]] — IPMI hash dump → crack → database credential extraction. `lab`

### Attacking Common Services

- [[labs/htb/attacking_common_services/dns_subdomain_enumeration_and_zone_transfer]] — subbrute against target nameserver → discover hr/helpdesk/ns subdomains → AXFR hr.inlanefreight.htb → TXT flag. `lab`
- [[labs/htb/attacking_common_services/smtp_user_enumeration_and_mail_access]] — smtp-user-enum RCPT mode finds marlin → Hydra cracks SMTP auth → IMAP telnet reads flag from inbox. `lab`
- [[labs/htb/attacking_common_services/easy_skill_assessment]] — Multi-service chain: SMTP enum → FTP brute (fiona) → Path A: CoreFTP CVE-2022-22836 PUT traversal → PHP webshell; Path B: MySQL OUTFILE → PHP webshell → RCE flag. `lab`
- [[labs/htb/attacking_common_services/medium_skill_assessment]] — DNS AXFR → internal FTP vHost (port 30021) → anonymous login → password list → POP3 brute → email with SSH key → shell. `lab`
- [[labs/htb/attacking_common_services/hard_skill_assessment]] — SMB null session → IT department files → NetExec brute Fiona → RDP → SQLCMD IMPERSONATE john → linked server → xp_cmdshell → Administrator flag. `lab`
- [[labs/htb/attacking_common_services/mssql_hash_theft_and_db_enumeration]] — xp_dirtree NTLMv2 hash theft → crack mssqlsvc password → flagDB schema walk → encrypted admin credential → flag. `lab`
- [[labs/htb/attacking_common_services/rdp_pass_the_hash]] — xfreerdp initial access → find NTLM hash → enable DisableRestrictedAdmin → PTH as Administrator → flag. `lab`

### AI Security

#### Prompt Injection Attacks

- [[labs/htb/prompt_injection_attacks/direct_prompt_injection]] — 8 direct injection strategies: rule amendment, storytelling, translation, spell-check, encoding, fragment exfiltration; business logic price manipulation. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/indirect_prompt_injection]] — Indirect injection via Discord CSV framing, URL/HTML comment (python3 http.server + tunnel), SMTP summarizer, application review bot. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/jailbreaks_1]] — Jailbreaks I: DAN (full v10.0 prompt + token-scarcity mechanic), grandma roleplay exploit, Bob & Alice fictional scenario. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/jailbreaks_2]] — Jailbreaks II: token smuggling (string concat, predict_mask), adversarial suffix (positive completion + computed), AntiGPT/opposite mode, IMM Haskell encoding with Python encode/decode helpers. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/mitigations]] — 7 defense layers (prompt engineering, blacklists, least privilege, human supervision, fine-tuning, adversarial training, guardrails) with bypass strategy per layer and detection signals. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/skills_assessment]] — Capstone: get CEO @vautia banned from HaWa Corp via indirect injection through surviving feature vectors (registration, reporting, profiles) when most features are disabled. `lab` `attack/ai`
- [[labs/htb/prompt_injection_attacks/reconnaissance_and_tools]] — LLM fingerprinting with llmmap (8-probe interactive workflow) and automated scanning with garak (dan.*, promptinject.* families). `lab` `attack/ai`

#### AI Evasion & Sparsity

- [[labs/htb/ai_evasion_jsma_challenge]] — JSMA challenge: fetch MNIST baseline, load LeNet-5 weights, implement saliency-guided pixel modification under L0 budget, submit via API. `lab` `attack/ai`

#### AI Data Attacks

- [[labs/htb/ai_data_attacks_label_flipping_challenge]] — OvR label flipping: poison Class 1 training labels, tune flip rate (20–35%), serialize and submit poisoned classifier via API. `lab` `attack/ai`

---

## Study Guides

- [[study_guide/coae]] — COAE exam prep: module-by-topic wiki coverage map, prioritised gap list, quick-reference checklists, Python attack snippets. `reference` `concept`
