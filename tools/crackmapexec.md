---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# crackmapexec (cme)

CrackMapExec is a post-exploitation and enumeration tool for SMB, WinRM, MSSQL, LDAP, and RDP — designed for authenticated and unauthenticated access testing across Windows networks.

## Overview

CrackMapExec (CME / cme) automates credential testing and enumeration against Windows protocols at scale. It can spray credentials across an IP range, enumerate shares and users, execute commands, and dump hashes — all from a single tool.

Install:
```bash
sudo apt install crackmapexec
# or
pip3 install crackmapexec
```

Note: NetExec (`nxc`) is the active fork/successor of CrackMapExec as of 2024. Syntax is identical; use `nxc` if `cme` is not available.

## Commands / Syntax

```bash
# SMB — enumerate shares (null session)
crackmapexec smb 10.129.14.128 --shares -u '' -p ''

# SMB — enumerate with credentials
crackmapexec smb 10.129.14.128 --shares -u 'user' -p 'password'

# SMB — enumerate users
crackmapexec smb 10.129.14.128 --users -u '' -p ''

# SMB — enumerate groups
crackmapexec smb 10.129.14.128 --groups -u '' -p ''

# SMB — enumerate logged-on users
crackmapexec smb 10.129.14.128 --loggedon-users

# SMB — spray credentials across a subnet
crackmapexec smb 10.129.14.0/24 -u users.txt -p passwords.txt

# SMB — pass-the-hash
crackmapexec smb 10.129.14.128 -u Administrator -H :NTHASH

# SMB — execute command
crackmapexec smb 10.129.14.128 -u user -p pass -x 'whoami'

# WinRM — test credentials
crackmapexec winrm 10.129.14.128 -u user -p pass

# MSSQL — enumerate
crackmapexec mssql 10.129.14.128 -u sa -p ''

# MSSQL — execute SQL query
crackmapexec mssql 10.129.14.128 -u sa -p '' --local-auth -q "SELECT @@version"
```

## Flags & Options

### Global Flags

| Flag | Description |
|------|-------------|
| `-u <user>` | Username or file of usernames |
| `-p <pass>` | Password or file of passwords |
| `-H <hash>` | NTLM hash (pass-the-hash) |
| `--local-auth` | Use local account (not domain) |
| `--continue-on-success` | Don't stop after first successful login |
| `-t <N>` | Number of threads (default 100) |
| `-v` | Verbose |

### SMB-Specific

| Flag | Description |
|------|-------------|
| `--shares` | Enumerate shares |
| `--users` | Enumerate users |
| `--groups` | Enumerate groups |
| `--loggedon-users` | Show logged-on users |
| `--rid-brute` | RID brute-force user enumeration |
| `--pass-pol` | Dump password policy |
| `-x <command>` | Execute shell command |
| `-X <command>` | Execute PowerShell command |
| `--sam` | Dump SAM database |
| `--lsa` | Dump LSA secrets |

## Reading Output

```
SMB  10.129.14.128  445  DEVSMB  [*] Windows 6.1 Build 0 (name:DEVSMB) (domain:) (signing:False) (SMBv1:False)
SMB  10.129.14.128  445  DEVSMB  [+] \:
SMB  10.129.14.128  445  DEVSMB  [+] Enumerated shares
SMB  10.129.14.128  445  DEVSMB  notes  READ,WRITE  CheckIT
```

- `[+]` = success
- `[-]` = failure
- `[*]` = informational
- `Pwn3d!` = user has admin privileges (appears after successful auth)
- `(signing:False)` = SMB signing not required → relay attacks possible

## Gotchas & Notes

- **`(Pwn3d!)`** means the authenticated user has local admin rights — immediately prioritize this host.
- **SMB signing check**: `(signing:False)` indicates the host is vulnerable to NTLM relay attacks. Note all hosts without signing.
- **Account lockout**: Always check the password policy before spraying. CME can dump policy with `--pass-pol`. Locked accounts end engagements.
- **NetExec**: CrackMapExec development moved to NetExec (`nxc`). Syntax is identical — replace `crackmapexec` with `nxc`.
- **`--continue-on-success`**: Without this flag, CME stops after the first valid credential match per host. Use it when spraying to find all valid creds.
- **Database**: CME stores results in a local database. Use `cme smb` followed by `cme -L` commands to query historical results.

## Related Pages

- [[enumeration/smb]]
- [[enumeration/mssql]]
- [[enumeration/windows_remote_mgmt]]
- [[tools/smbclient]]
- [[tools/impacket]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
