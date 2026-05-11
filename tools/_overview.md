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
| [[tools/nmap]] | Port scan, service detect, NSE scripts | Yes (SYN scan) | Low by default; evasion flags available |
| masscan | Bulk port sweep (millions/sec) | Yes | Very low |
| netcat | Banner grabbing, manual probing | No | Moderate |

### DNS enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/dig]] | DNS queries, zone transfers | Gold standard for manual DNS work |
| [[tools/dnsenum]] | Full DNS recon in one run | Combines NS/MX/AXFR/brute-force |
| fierce | Subdomain discovery | Good for permutation-based recon |
| gobuster (dns mode) | Fast subdomain brute-force | Parallelized; needs wordlist |

### SMB / Windows enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/smbclient]] | Browse shares, download files | Interactive or one-shot |
| [[tools/enum4linux]] | Users, groups, shares, policies | Wrapper around smbclient + rpcclient |
| [[tools/rpcclient]] | RPC enumeration, RID brute-force | Best for user/group enumeration |
| [[tools/crackmapexec]] | Auth testing, pass-the-hash, spray | Multi-protocol (SMB/WinRM/MSSQL) |
| [[tools/impacket]] | Full Windows attack suite | Python; secretsdump, psexec, wmiexec |

### SNMP enumeration

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/onesixtyone]] | Community string brute-force | Fast; use before snmpwalk |
| [[tools/snmpwalk]] | Walk MIB tree | Needs valid community string |
| braa | Bulk SNMP queries | Faster than snmpwalk for large targets |

### Database tools

| Tool | Primary use | Protocols |
|------|------------|-----------|
| [[tools/impacket]] | MSSQL via mssqlclient.py | MSSQL |
| [[tools/odat]] | Oracle TNS attack suite | Oracle |
| mysql client | MySQL interactive queries | MySQL |
| sqlplus | Oracle interactive queries | Oracle |

### Privilege escalation (Linux)

| Tool | Primary use | Notes |
|------|------------|-------|
| [[tools/linpeas]] | Automated LPE enumeration | Color-coded by confidence |
| [[tools/pspy]] | Process spy without root | Catches cron jobs and credential leaks |
| GTFOBins | SUID/sudo escape reference | Web resource, not a CLI tool |

## Tool selection decision tree

```
Starting enumeration on a host?
  └─ Unknown open ports → [[tools/nmap]] -sV -sC
  └─ SMB open (445) → [[tools/enum4linux]] then [[tools/crackmapexec]]
  └─ DNS open (53) → [[tools/dig]] AXFR, then [[tools/dnsenum]]
  └─ SNMP open (161) → [[tools/onesixtyone]] then [[tools/snmpwalk]]
  └─ MSSQL (1433) → [[tools/nmap]] scripts, then [[tools/impacket]] mssqlclient
  └─ Oracle (1521) → [[tools/odat]] sidguesser, then sqlplus

Got a shell, need to escalate (Linux)?
  └─ Start with [[tools/linpeas]] for overview
  └─ Watch processes with [[tools/pspy]]
  └─ Check sudo -l → [[attack/linux_privesc_sudo_suid]]
  └─ Check SUID → [[attack/linux_privesc_sudo_suid]]
```

## Related pages

- [[tools/nmap]]
- [[tools/crackmapexec]]
- [[tools/impacket]]
- [[tools/linpeas]]
- [[enumeration/_overview]]
- [[attack/linux_privilege_escalation]]
