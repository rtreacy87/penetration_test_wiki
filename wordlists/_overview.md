---
tags: [wordlist, concept]
module: multi
last_updated: 2026-05-10
source_count: 0
---

# Wordlists Overview

Reference guide for choosing and generating wordlists for brute-force and directory/subdomain enumeration.

## Overview

A wordlist is only as good as its match to the target context. Generic lists waste time; targeted or mutated lists find results that generic ones miss.

## SecLists — primary collection

SecLists (`/usr/share/seclists/` on Kali) is the standard collection. Key subdirectories:

| Directory | Contents | Use case |
|-----------|----------|----------|
| `Discovery/Web-Content/` | Directory and file names | Web directory brute-force |
| `Discovery/DNS/` | Subdomain names | DNS brute-force |
| `Usernames/` | Username lists | User enumeration, login brute-force |
| `Passwords/` | Password lists | Auth brute-force |
| `Fuzzing/` | Special chars, payloads | Injection testing |
| `Miscellaneous/` | Misc lists | Various |

## Wordlist selection by task

See [[wordlists/use_cases]] for a full mapping of task → recommended wordlist.

Quick reference:

| Task | Recommended list |
|------|-----------------|
| Web directory scan (fast) | `Discovery/Web-Content/common.txt` |
| Web directory scan (thorough) | `Discovery/Web-Content/raft-large-directories.txt` |
| Subdomain brute-force | `Discovery/DNS/subdomains-top1million-5000.txt` |
| SMB/RID brute-force | `Usernames/xato-net-10-million-usernames.txt` |
| Password spray | `Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| SSH/service brute-force | `Passwords/Leaked-Databases/rockyou.txt` |
| SNMP community strings | `Discovery/SNMP/common-snmp-community-strings.txt` |

## Custom wordlist generation

### CeWL — website wordlist

```bash
# Scrape target website for custom words (depth 2, min length 5)
cewl http://target.com -d 2 -m 5 -w custom.txt
```

### Crunch — pattern-based generation

```bash
# Generate 8-char passwords: lowercase + digits
crunch 8 8 abcdefghijklmnopqrstuvwxyz0123456789 -o output.txt

# Pattern: Company + 4 digits
crunch 11 11 -t Company%%%% -o company_passwords.txt
```

### Hashcat rules — mutate existing lists

```bash
# Apply best64 rules to rockyou
hashcat --stdout rockyou.txt -r /usr/share/hashcat/rules/best64.rule > mutated.txt
```

### Username generation from employee names

```bash
# Use username-anarchy with a names file
username-anarchy -f names.txt > usernames.txt
```

## Related pages

- [[wordlists/use_cases]]
- [[enumeration/_overview]]
- [[tools/nmap]]
