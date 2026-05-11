---
tags: [lab, concept]
module: multi
last_updated: 2026-05-10
source_count: 3
---

# Labs Overview

Index of lab write-ups organized by platform and difficulty.

## Overview

Lab write-ups document actual engagement steps against practice targets. They serve as both a reference for similar real-world scenarios and a record of techniques that worked in context. Every write-up links back to the relevant technique and tool pages.

## HackTheBox (HTB)

### Footprinting labs

| Lab | Difficulty | Key techniques |
|-----|-----------|----------------|
| [[labs/htb/footprinting_easy]] | Easy | DNS zone transfer, NFS SSH key extraction |
| [[labs/htb/footprinting_medium]] | Medium | FTP, SMB, NFS, SNMP multi-service chaining |
| [[labs/htb/footprinting_hard]] | Hard | IPMI hash dump, crack, database credential extraction |

## TryHackMe (THM)

_(write-ups to be added)_

## Write-up structure

Each lab write-up follows this template:

- **Target info** — OS, difficulty, IP range
- **Recon steps** — commands run and findings
- **Enumeration findings** — services, versions, misconfigs discovered
- **Exploitation path** — how initial access was obtained
- **Post-exploitation** — privilege escalation, flags
- **Lessons learned** — what was non-obvious or worth remembering
- **Links** to relevant technique and tool pages

## Related pages

- [[recon/_overview]]
- [[enumeration/_overview]]
- [[tools/_overview]]
