# Wiki Index

_Last updated: 2026-05-11 — 86 pages total_

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
- [[enumeration/smb]] — SMB versions, null sessions, rpcclient RID brute-force, smbclient, enum4linux, CrackMapExec. `enumeration` `enumeration/smb`
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
- [[attack/ai/data_poisoning]] — Hub: AI pipeline attack surface, OWASP LLM03/LLM05 mapping, all data attack types. `attack` `attack/ai` `concept`
- [[attack/ai/label_flipping]] — Label flipping and targeted label attacks: decision boundary math, flip_labels implementation, evaluation. `attack` `attack/ai`
- [[attack/ai/clean_label_attacks]] — Feature-perturbation poisoning: target selection, perturbation vector, one-step boundary shift without touching labels. `attack` `attack/ai`
- [[attack/ai/trojan_attacks]] — Backdoor attacks: trigger embedding, CNN architecture, Clean Accuracy vs ASR (100% on GTSRB Stop→Speed 60). `attack` `attack/ai`
- [[attack/ai/model_steganography]] — Pickle `__reduce__` RCE + tensor LSB steganography: full reverse shell chain via malicious `.pth` file. `attack` `attack/ai`

---

## Tools

- [[tools/_overview]] — Tool selection guide: comparison tables by function and decision tree by scenario. `tool` `concept`
- [[tools/nmap]] — Port scanning, service detection, NSE scripts, OS detection, performance, firewall evasion. `tool`
- [[tools/crackmapexec]] — SMB/WinRM/MSSQL enumeration, auth testing, pass-the-hash, password spray. `tool`
- [[tools/impacket]] — Suite: mssqlclient, psexec, secretsdump, wmiexec, GetUserSPNs, samrdump. `tool`
- [[tools/smbclient]] — SMB share browsing, file operations, null session syntax. `tool`
- [[tools/enum4linux]] — enum4linux-ng: SMB/NetBIOS user/group/share enumeration. `tool`
- [[tools/rpcclient]] — RPC enumeration commands, RID brute-force, queryuser/enumdomusers. `tool`
- [[tools/snmpwalk]] — MIB tree walking, key OID subtrees, community string usage. `tool`
- [[tools/onesixtyone]] — SNMP community string brute-force; use before snmpwalk. `tool`
- [[tools/dig]] — DNS query tool: all record types, AXFR syntax, output format, bash brute-force loop. `tool`
- [[tools/dnsenum]] — DNS enumeration: NS/MX/AXFR/brute-force in one run. `tool`
- [[tools/odat]] — Oracle attack tool: SID enum, auth brute-force, web shell upload. `tool`
- [[tools/linpeas]] — Automated Linux privilege escalation enumeration; color-coded output. `tool`
- [[tools/pspy]] — Unprivileged process monitor; catches cron and credential leaks. `tool`
- [[tools/responder]] — LLMNR/NBT-NS poisoner; captures NTLMv2 hashes from Windows broadcast auth. `tool`
- [[tools/medusa]] — Parallel brute-forcer; more reliable than Hydra for FTP/SSH. `tool`
- [[tools/hydra]] — Versatile network login brute-forcer; 50+ protocols, go-to for HTTP/RDP/SMTP/POP3. `tool`
- [[tools/crowbar]] — RDP-focused brute-forcer; more reliable than Hydra for RDP password spraying. `tool`

---

## Definitions

- [[definitions/_overview]] — Index of all definition pages: protocols, flags, auth, security terms. `definition` `concept`
- [[definitions/network_protocols]] — TCP, UDP, ICMP, SCTP, ARP, QUIC — behavior and pentester relevance for each. `definition` `concept`
- [[definitions/tcp_flags]] — SYN/ACK/FIN/RST/PSH/URG — three-way handshake, port state inference, scan type selection. `definition` `reference`
- [[definitions/auth_protocols]] — NTLM, Kerberos, LDAP, OAuth, SAML, JWT — how they work and how they fail. `definition` `concept`
- [[definitions/security_terminology]] — CVE/CVSS, RCE, LFI, SSRF, IDOR, SQLi, XSS, SSTI, lateral movement, persistence, defense terms. `definition` `concept`

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
- [[labs/htb/footprinting_easy]] — DNS zone transfer + NFS SSH key extraction → flag. `lab`
- [[labs/htb/footprinting_medium]] — Multi-service (FTP/SMB/NFS/SNMP) chaining to obtain credentials. `lab`
- [[labs/htb/footprinting_hard]] — IPMI hash dump → crack → database credential extraction. `lab`
