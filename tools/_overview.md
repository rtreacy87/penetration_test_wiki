---
tags: [tool, concept]
module: multi
last_updated: 2026-05-10
source_count: 6
---

# Tools Overview

Quick-reference comparison of tools used across the penetration testing lifecycle.

## Overview

Tools are organized by primary function. Most tools appear in enumeration or exploitation workflows; this page exists to answer "which tool do I reach for?" at each stage.

## Tool comparison by function

### Network scanning

| Tool | Primary use | Requires root | Stealth |
|------|------------|---------------|---------|
| [[tools/enumeration/nmap]] | Port scan, service detect, NSE scripts | Yes (SYN scan) | Low by default; evasion flags available |
| masscan | Bulk port sweep (millions/sec) | Yes | Very low |
| netcat | Banner grabbing, manual probing | No | Moderate |

### DNS enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/enumeration/dig]] | DNS queries, zone transfers | Gold standard for manual DNS work |
| [[tools/enumeration/dnsenum]] | Full DNS recon in one run | Combines NS/MX/AXFR/brute-force |
| fierce | Subdomain discovery | Good for permutation-based recon |
| gobuster (dns mode) | Fast subdomain brute-force | Parallelized; needs wordlist |

### SMB / Windows enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/enumeration/smbclient]] | Browse shares, download files | Interactive or one-shot |
| [[tools/enumeration/enum4linux]] | Users, groups, shares, policies | Wrapper around smbclient + rpcclient |
| [[tools/enumeration/rpcclient]] | RPC enumeration, RID brute-force | Best for user/group enumeration |
| [[tools/attack/netexec]] | Credential spray, validation, pass-the-hash | Multi-protocol (SMB/WinRM/MSSQL); wrong tool for large-file brute-force |
| [[tools/utility/impacket]] | Full Windows attack suite | Python; secretsdump, psexec, wmiexec |
| [[tools/attack/metasploit]] | SMB brute-force, exploitation, post-exploitation | Use smb_login for rockyou-scale attacks; CME for spraying |

### SNMP enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/enumeration/onesixtyone]] | Community string brute-force | Fast; use before snmpwalk |
| [[tools/enumeration/snmpwalk]] | Walk MIB tree | Needs valid community string |
| braa | Bulk SNMP queries | Faster than snmpwalk for large targets |

### Database tools

| Tool | Primary use | Protocols |
|------|------------|-----------|
| [[tools/utility/impacket]] | MSSQL via mssqlclient.py | MSSQL |
| [[tools/attack/odat]] | Oracle TNS attack suite | Oracle |
| mysql client | MySQL interactive queries | MySQL |
| sqlplus | Oracle interactive queries | Oracle |

### Privilege escalation (Linux)

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/enumeration/linpeas]] | Automated LPE enumeration | Color-coded by confidence |
| [[tools/enumeration/pspy]] | Process spy without root | Catches cron jobs and credential leaks |
| GTFOBins | SUID/sudo escape reference | Web resource, not a CLI tool |

## Tool selection decision tree

```
Starting enumeration on a host?
  └─ Unknown open ports → [[tools/enumeration/nmap]] -sV -sC
  └─ SMB open (445) → [[tools/enumeration/enum4linux]] then [[tools/attack/netexec]]
  └─ DNS open (53) → [[tools/enumeration/dig]] AXFR, then [[tools/enumeration/dnsenum]]
  └─ SNMP open (161) → [[tools/enumeration/onesixtyone]] then [[tools/enumeration/snmpwalk]]
  └─ MSSQL (1433) → [[tools/enumeration/nmap]] scripts, then [[tools/utility/impacket]] mssqlclient
  └─ Oracle (1521) → [[tools/attack/odat]] sidguesser, then sqlplus

Need to brute-force credentials?
  └─ SMB, large wordlist (rockyou) → [[tools/attack/metasploit]] smb_login (STOP_ON_SUCCESS)
  └─ SMB, short list / spray → [[tools/attack/netexec]] (--no-bruteforce --continue-on-success)
  └─ SSH / FTP / HTTP → [[tools/attack/hydra]] (fastest for non-SMB)
  └─ Multiple protocols from one tool → [[tools/attack/metasploit]] auxiliary/scanner/*_login

Got a shell, need to escalate (Linux)?
  └─ Start with [[tools/enumeration/linpeas]] for overview
  └─ Watch processes with [[tools/enumeration/pspy]]
  └─ Check sudo -l → [[attack/linux_privesc_sudo_suid]]
  └─ Check SUID → [[attack/linux_privesc_sudo_suid]]
```

## Related pages

- [[tools/enumeration/nmap]]
- [[tools/attack/netexec]]
- [[tools/utility/impacket]]
- [[tools/enumeration/linpeas]]
- [[enumeration/_overview]]
- [[attack/linux_privilege_escalation]]
