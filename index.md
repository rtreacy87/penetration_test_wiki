# Wiki Index

_Last updated: 2026-05-10 — 57 pages total_

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

- [[attack/_overview]] — Map of all attack categories: network, Linux privesc, web, AI/ML. `attack` `concept`

### Linux Privilege Escalation

- [[attack/linux_privilege_escalation]] — Hub page: all LPE categories, quick-wins checklist, methodology flow. `attack` `attack/privesc`
- [[attack/linux_privesc_enumeration]] — Post-exploitation enumeration: OS, users, network, files, services, creds, procfs. `attack` `attack/privesc`
- [[attack/linux_privesc_sudo_suid]] — Sudo abuse, SUID/SGID exploitation, capabilities, privileged group abuse (docker, lxd). `attack` `attack/privesc`
- [[attack/linux_privesc_services]] — Cron job abuse, Docker escape, LXD, Kubernetes, logrotate, Screen, NFS, tmux. `attack` `attack/privesc`
- [[attack/linux_privesc_kernel]] — Dirty Pipe, Netfilter CVEs, PwnKit/Polkit, Baron Samedit, library hijacking, Python hijacking. `attack` `attack/privesc`

### Network Attacks

- [[attack/network/firewall_evasion]] — Nmap-based firewall/IDS evasion: fragmentation, decoys, source port, timing. `attack` `attack/network`

### AI / LLM Attacks

- [[attack/ai/_overview]] — Full map of AI attack surface: injection, jailbreaking, adversarial ML, MCP, system attacks. `attack` `attack/ai` `concept`
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
