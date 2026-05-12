---
tags: [tool, enumeration/smtp]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# smtp-user-enum

SMTP user enumeration tool; tests whether usernames exist on a mail server by probing the VRFY, EXPN, or RCPT TO commands without requiring authentication.

## Overview

smtp-user-enum automates the process of sending one of three SMTP commands for each candidate username in a wordlist and recording which ones the server confirms as valid. It is the standard tool for SMTP user enumeration when you have a target mail server and want to produce a confirmed user list for follow-on credential attacks.

**Use smtp-user-enum when:**
- You have access to an SMTP server (TCP/25 or 587) and want to identify valid accounts before attempting brute-force
- You need to enumerate mail users on an internal server where directory services (LDAP, RPC) are not accessible
- You want to confirm a specific username exists before targeting it with Hydra or Medusa

The confirmed output feeds directly into tools like Hydra (SMTP AUTH brute-force) or Metasploit modules.

## Installation

```bash
# Check if installed
smtp-user-enum --help 2>/dev/null | head -3 || echo "not installed"

# Install (Kali / Parrot)
sudo apt install smtp-user-enum -y

# Verify
smtp-user-enum --help
```

---

## Usage

### RCPT TO mode (recommended)

```bash
smtp-user-enum -M RCPT -U users.txt -D example.com -t 10.129.x.x
```

Tests `RCPT TO:<user@example.com>` for each entry in `users.txt`. A `250 OK` response means the user exists; `550` or similar means they don't.

### VRFY mode

```bash
smtp-user-enum -M VRFY -U users.txt -t 10.129.x.x
```

Uses the `VRFY` command, which directly asks the server to verify a username. Faster and cleaner when available, but commonly disabled.

### EXPN mode

```bash
smtp-user-enum -M EXPN -U users.txt -t 10.129.x.x
```

Uses the `EXPN` command to expand mailing list aliases. Rarely available but occasionally reveals group membership.

### Single username check

```bash
smtp-user-enum -M VRFY -u root -t 10.129.x.x
smtp-user-enum -M RCPT -u admin -D example.com -t 10.129.x.x
```

Useful for quickly confirming whether a specific account exists before running a full wordlist.

### Non-standard port

```bash
smtp-user-enum -M RCPT -U users.txt -D example.com -t 10.129.x.x -p 587
```

---

## Flags & Options

| Flag | Description |
|------|-------------|
| `-M <mode>` | Enumeration method: `VRFY`, `EXPN`, or `RCPT` |
| `-U <file>` | Wordlist of usernames to test (one per line) |
| `-u <user>` | Test a single username instead of a wordlist |
| `-D <domain>` | Domain to append to usernames — constructs `user@domain` format for RCPT mode |
| `-t <host>` | Target SMTP server IP or hostname |
| `-p <port>` | Target port (default: 25) |
| `-w <secs>` | Timeout per query in seconds (default: 5) |
| `-v` | Verbose — show each probe and response |
| `-o <file>` | Write results to a file |

---

## Choosing the right mode

| Mode | When to use | Why |
|------|-------------|-----|
| `RCPT` | Default choice — try this first | Works even when VRFY and EXPN are disabled; uses the delivery mechanism that must exist for the server to function |
| `VRFY` | When RCPT gives inconsistent results or you confirm VRFY is enabled | Faster, more explicit; directly answers "does this user exist?" |
| `EXPN` | Only try if you suspect mailing list aliases are in scope | Almost never enabled on modern servers |

Most production mail servers disable VRFY (`disable_vrfy_command = yes` in Postfix) because it leaks user information. RCPT is harder to fully disable without breaking mail routing, making it the most reliable method.

### How to confirm which methods are available

Connect manually and test before running the full enumeration:

```bash
telnet 10.129.x.x 25

EHLO test
VRFY root               # 252 or 550 = disabled/no user; 250 = enabled and found
EXPN postmaster         # 550 = disabled; 250 = enabled
```

If VRFY returns `502 Command not implemented` or `252 Cannot VRFY user`, switch to RCPT mode.

---

## Interpreting output

```
10.129.x.x: marlin@inlanefreight.htb exists
```

A line like this means the server returned `250 OK` for that address. All other addresses returned `550` (no such user) or similar rejection codes and are not shown.

The summary at the end shows total queries, runtime, and query rate — useful for gauging whether the server is rate-limiting.

---

## Gotchas & Notes

- **Use `-D` for RCPT mode** — without the domain flag, smtp-user-enum sends `RCPT TO:<user>` without a domain, which most servers reject entirely
- **Response code `252`** is ambiguous — it means the server cannot verify the user but will attempt delivery anyway; some tools treat it as confirmed, smtp-user-enum does not count it as a hit by default
- **Rate limiting** — aggressive probing (default 5 workers) can trigger rate limiting or temporary blocks on hardened servers; reduce with careful timeout tuning or run serially
- **Full email format vs username only** — when you pass the output of smtp-user-enum to Hydra for SMTP AUTH brute-force, Hydra's `-l` flag needs `user@domain`, not just `user`; construct it from the username + domain accordingly

## Related Pages

- [[enumeration/smtp]] — SMTP banner grabbing, manual VRFY/EXPN/RCPT probing, nmap scripts
- [[attack/smtp]] — full attack reference: user enum, O365 spray, open relay, OpenSMTPD RCE
- [[tools/attack/hydra]] — follow-on brute-force after confirming valid usernames
- [[labs/htb/attacking_common_services/smtp_user_enumeration_and_mail_access]] — lab using smtp-user-enum + Hydra + IMAP

## Sources

- raw/lab/attacking_common_services/attacking_email.md
