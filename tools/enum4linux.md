---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# enum4linux / enum4linux-ng

enum4linux-ng is an SMB/NetBIOS enumeration tool that automates querying Windows and Samba servers for users, groups, shares, password policies, and OS information.

## Overview

enum4linux-ng is the modern rewrite of the original enum4linux (Perl). Written in Python, it performs a comprehensive set of queries against SMB/RPC services in a single run. It outputs structured data (YAML/JSON supported) and is more reliable than the original.

Install:
```bash
git clone https://github.com/cddmp/enum4linux-ng.git
cd enum4linux-ng
pip3 install -r requirements.txt
```

## Commands / Syntax

```bash
# Full enumeration (all checks)
./enum4linux-ng.py 10.129.14.128 -A

# Output as YAML
./enum4linux-ng.py 10.129.14.128 -A -oY output.yaml

# Output as JSON
./enum4linux-ng.py 10.129.14.128 -A -oJ output.json

# With credentials
./enum4linux-ng.py 10.129.14.128 -A -u username -p password

# Null session only
./enum4linux-ng.py 10.129.14.128 -A -u '' -p ''
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-A` | All checks |
| `-U` | Users |
| `-G` | Groups |
| `-S` | Shares |
| `-C` | Computers |
| `-P` | Password policy |
| `-N` | NetBIOS names |
| `-R` | RID cycling (user enumeration via RIDs) |
| `-u <user>` | Username |
| `-p <pass>` | Password |
| `-oY <file>` | YAML output |
| `-oJ <file>` | JSON output |
| `-v` | Verbose |
| `--timeout <N>` | Connection timeout |

## What It Enumerates

| Category | Data Returned |
|----------|-------------|
| NetBIOS names | Machine name, workgroup, services |
| SMB dialect | SMB1/2/3 support, signing requirements |
| RPC session | Whether null session is allowed |
| Domain info | Domain name, SID, domain controller status |
| OS info | OS version and type |
| Users | Usernames, RIDs, account flags |
| Groups | Local, builtin, and domain groups |
| Shares | Share names, types, permissions (mapping/listing test) |
| Password policy | Minimum length, complexity, lockout settings |
| Printers | Network printers |

## Gotchas & Notes

- Run enum4linux-ng in addition to (not instead of) manual rpcclient and smbclient. It automates many checks but may miss things.
- The `-R` flag performs RID cycling — more comprehensive than just `enumdomusers`.
- Password policy output reveals lockout thresholds — check before brute-forcing.
- "Random user session" message means the server allows sessions with arbitrary usernames — run again with a random user for potentially more results.
- The original `enum4linux` (Perl) is available in Kali by default but produces less reliable output. Prefer enum4linux-ng when possible.

## Related Pages

- [[enumeration/smb]]
- [[tools/smbclient]]
- [[tools/rpcclient]]
- [[tools/netexec]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
