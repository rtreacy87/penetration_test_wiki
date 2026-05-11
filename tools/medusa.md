---
tags: [tool, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 1
---

# Medusa

Parallel network login auditor; supports FTP, SSH, HTTP, SMB, MSSQL, POP3, IMAP, and more.

## Overview

Medusa is a fast, modular brute-force tool similar to Hydra but with better parallelism for single-service attacks. Preferred over Hydra for FTP and SSH due to more reliable module implementations.

## Installation

```bash
apt install medusa
```

## Common usage

```bash
# Single user, password list
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp

# User list, password list
medusa -U users.txt -P passwords.txt -h 10.129.203.7 -M ftp

# SSH
medusa -u admin -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ssh
```

## Common flags

| Flag | Description |
|------|-------------|
| `-h <host>` | Target host |
| `-H <file>` | Target host list |
| `-u <user>` | Single username |
| `-U <file>` | Username list |
| `-p <pass>` | Single password |
| `-P <file>` | Password list |
| `-M <module>` | Service module (ftp, ssh, smb, http, etc.) |
| `-t <n>` | Parallel login attempts (default 16) |
| `-f` | Stop after first valid credential found |
| `-n <port>` | Override default port |

## Supported modules

```bash
medusa -d    # list all available modules
```

Common modules: `ftp`, `ssh`, `smb`, `http`, `https`, `imap`, `pop3`, `mssql`, `mysql`, `rdp`, `telnet`.

## Gotchas & notes

- `-f` stops immediately on success; omit it when you want all valid credentials
- FTP brute force may be slower against servers with connection-rate limiting; reduce with `-t 2`
- Compare to [[tools/hydra]] — Hydra is more commonly shown in HTB walkthroughs but Medusa handles FTP more reliably

## Related pages

- [[attack/ftp]]
- [[tools/hydra]]
- [[wordlists/use_cases]]

## Sources

- raw/attacking_common_services/ftp_attacking_ftp.md
