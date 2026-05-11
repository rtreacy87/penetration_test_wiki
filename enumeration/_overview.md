---
tags: [enumeration, concept]
module: footprinting
last_updated: 2026-05-10
source_count: 2
---

# Enumeration Overview

What enumeration is, the six-layer model that structures it, and the methodology for working through each layer systematically.

## Overview

Enumeration is information gathering using both active (scanning, direct queries) and passive (third-party providers) methods, applied to a known target. It is a loop: each piece of information found opens avenues for collecting more. It operates on domains, IP addresses, and accessible services.

Unlike pure OSINT, enumeration involves active contact with the target. The two are complementary but must not be conflated — OSINT first, enumeration second.

The core mistake most testers make: seeing SSH, RDP, or WinRM and immediately attempting brute-force. This is noisy, can result in blacklisting, and skips layers of understanding that could reveal a far easier path.

## The Six-Layer Model

A static methodology that imposes structure on a dynamic process. Each layer represents a boundary to be studied and ultimately passed.

| Layer | Focus | What to Find |
|-------|-------|-------------|
| 1. Internet Presence | External footprint | Domains, subdomains, vHosts, ASN, netblocks, IPs, cloud instances |
| 2. Gateway | Perimeter defenses | Firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, VPN, Cloudflare |
| 3. Accessible Services | Exposed interfaces | Service type, functionality, config, port, version |
| 4. Processes | Service internals | PID, tasks, data flows, source/destination |
| 5. Privileges | Access controls | Groups, users, permissions, restrictions, environment |
| 6. OS Setup | Host internals | OS type, patch level, network config, sensitive files |

Layers 1–2 are primarily addressed during recon. The service-level pages in this wiki address Layers 3–6.

## Infrastructure-Based vs. Host-Based Enumeration

**Infrastructure-based** (Layers 1–2):
- Domain/subdomain discovery
- Cloud resource discovery (S3, Azure blobs, GCP)
- Staff and technology fingerprinting

**Host-based** (Layers 3–6):
- Port scanning and service identification
- Protocol-specific enumeration (DNS, FTP, SMB, SNMP, etc.)
- Authentication probing

**OS-based** (Layers 5–6):
- Privilege enumeration
- Configuration file inspection
- Sensitive data discovery

## Methodology vs. Cheat Sheet

A methodology is a systematic approach, not a step-by-step recipe. Tools and commands are the cheat sheet — the methodology is the structure that tells you *why* you're running them. The tool choices change; the layers do not.

Key implication: the order of tools is dynamic. Use multiple tools for the same service; each returns different output. Never rely solely on a single automated tool.

## The Enumeration Loop

Enumeration is not linear. It is a loop:
1. Discover a service or resource
2. Enumerate it for details
3. Use those details to discover more services or resources
4. Return to step 2

Each iteration builds on the last. A username found in SNMP may unlock FTP. A hostname from a zone transfer may expose a new service to enumerate.

## Core Principles (Repeated from Methodology)

1. There is more than meets the eye. Consider all points of view.
2. Distinguish between what we see and what we do not see.
3. There are always ways to gain more information. Understand the target.

## Gotchas & Notes

- Automated tools have blind spots. Always verify results manually.
- Some services deliberately return misleading responses (VRFY on SMTP returning 252 regardless of whether the user exists).
- A four-week penetration test cannot exhaustively enumerate a large environment. Prioritize breadth first, then depth on high-value targets.
- Internal infrastructure (Active Directory, intranet) follows the same six-layer model but is typically covered in separate modules.

## Related Pages

- [[recon/_overview]]
- [[enumeration/dns]]
- [[enumeration/ftp]]
- [[enumeration/smtp]]
- [[enumeration/smb]]
- [[enumeration/snmp]]
- [[enumeration/nfs]]
- [[enumeration/imap_pop3]]
- [[enumeration/ipmi]]
- [[enumeration/mssql]]
- [[enumeration/mysql]]
- [[enumeration/oracle_tns]]
- [[enumeration/linux_remote_mgmt]]
- [[enumeration/windows_remote_mgmt]]

## Sources

- raw/footprinting/introduction_enumeration_principles.md
- raw/footprinting/introduction_enumeration_methodology.md
