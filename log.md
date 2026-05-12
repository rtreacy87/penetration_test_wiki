# Wiki Log

## [2026-05-11] update | Expanded attack/sql_databases.md — connecting commands and interactive prompt guide
Pages updated: 1 (attack/sql_databases.md).
Key additions: Connecting section rewritten — each tool (mssqlclient.py, sqsh, sqlcmd, mysql) is a separate subsection with flag-by-flag explanations for beginners; mssqlclient.py target format corrected to `user:password@host`. New "Using the SQL prompt" section added explaining the mssqlclient.py (immediate) vs sqsh (batch/GO) execution models, common mistakes (forgetting GO, typing shell commands without xp_cmdshell), sqsh meta-commands, and a first-login checklist (SYSTEM_USER, IS_SRVROLEMEMBER, list databases, switch databases, list tables).

## [2026-05-11] update | Created tools/netexec.md; replaced CrackMapExec with NetExec across wiki
Pages created: 1 (tools/netexec.md). Pages updated: 16 (enumeration/smb, attack/smb, attack/rdp, attack/sql_databases, enumeration/windows_remote_mgmt, enumeration/mssql, definitions/auth_protocols, labs/htb/footprinting_medium, tools/crackmapexec, tools/_overview, tools/metasploit, tools/responder, tools/enum4linux, tools/impacket, tools/smbclient, tools/rpcclient, wordlists/use_cases, index.md).
Key additions: netexec.md is a full tool page covering all nxc protocols (SMB/WinRM/MSSQL/LDAP/SSH/RDP/FTP/VNC), enumeration, spraying, PTH, command execution, hash dumping, AS-REP roasting, Kerberoasting, and full flags reference. crackmapexec.md demoted with LEGACY header pointing to netexec. All `crackmapexec smb/winrm/mssql` commands replaced with `nxc smb/winrm/mssql`. All `[[tools/crackmapexec]]` wikilinks replaced with `[[tools/netexec]]` except the legacy page entry in index.md.

## [2026-05-11] update | Added tools/metasploit.md; clarified MSF vs CME for SMB brute-force
Pages created: 1 (tools/metasploit.md). Pages updated: 4 (tools/crackmapexec, tools/_overview, wordlists/use_cases, index.md).
Key additions: metasploit.md covers msfconsole module workflow (use/set/run), smb_login/ssh_login/ftp_login brute-force with STOP_ON_SUCCESS, discovery modules, and the MSF vs CME vs Hydra decision table. crackmapexec.md updated with a dedicated "CME vs MSF" section explaining why CME fails for large wordlist brute-force (no stop_on_success, unreliable with 14M-entry files, gzipped rockyou is the most common failure cause). wordlists/use_cases.md SMB section expanded with per-tool guidance and a note diagnosing the gzipped rockyou failure. tools/_overview added a credential brute-force decision subtree.

## [2026-05-11] update | Added GitHub repo links to wordlist pages; fixed rockyou.txt.gz issue
Pages updated: 2 (wordlists/_overview, wordlists/use_cases).
Key additions: Corrected rockyou.txt entry to show it ships as .gz on Kali/Parrot and requires gunzip; added three extraction options (gunzip from apt package, extract from seclists apt package, extract from git clone). Added GitHub links to all wordlist sources — SecLists, PayloadsAllTheThings, FuzzDB, Assetnote (github.com/assetnote/wordlists), Kaonashi, Probable-Wordlists, Rockyou2021, OneRuleToRuleThemAll (NotSoSecure). Added prominent warning callout in use_cases.md about rockyou.txt.gz needing decompression before use. Fixed SecLists table to show rockyou ships as .tar.gz inside SecLists.

## [2026-05-11] update | Expanded wordlist section — sources, install paths, per-tool recommendations
Pages updated: 2 (wordlists/_overview, wordlists/use_cases).
Key additions: _overview now covers installation paths for Kali and Parrot (SecLists, rockyou, dirb, dirbuster, wfuzz, Metasploit, John), web sources (SecLists GitHub, PayloadsAllTheThings, FuzzDB, Assetnote, CrackStation, WeakPass, Probable-Wordlists, Rockyou2021), and download commands. use_cases expanded to cover every major tool (Hydra, Medusa, CrackMapExec, Gobuster, ffuf, Feroxbuster, dnsenum/fierce, onesixtyone, Hashcat/John, Metasploit auxiliary modules, Nmap NSE brute scripts, WPScan) with per-service wordlist tables (FTP username+password lists, SSH, SMTP, SMB, SNMP, HTTP, MySQL, MSSQL, RDP, IPMI, Oracle), and a dedicated default credential files section with -C usage.

## [2026-05-11] update | Added Version Detection & Exploit Research sections to all enumeration pages
Pages updated: 13 (dns, ftp, imap_pop3, ipmi, linux_remote_mgmt, mssql, mysql, nfs, oracle_tns, smb, smtp, snmp, windows_remote_mgmt).
Key additions: Each page now has a dedicated "Version Detection & Exploit Research" section covering: how to extract the version/banner pre-authentication, a table of version-detection commands, searchsploit/Metasploit search syntax, and a Notable CVEs table with CVE numbers, affected versions, and impact. Highlights include BlueKeep/DejaBlue (RDP), EternalBlue/SMBGhost/SambaCry (SMB), regreSSHion CVE-2024-6387 (SSH), OpenSMTPD CVE-2020-7247 (SMTP), Exim CVE-2019-10149 (SMTP), vsFTPd backdoor CVE-2011-2523 (FTP), MySQL CVE-2012-2122 auth bypass, TNS Poison CVE-2012-1675 (Oracle), IPMI RAKP design flaw.

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
