---
tags: [recon, concept]
module: footprinting
last_updated: 2026-05-10
source_count: 2
---

# Recon Overview

High-level methodology for reconnaissance: the difference between passive (OSINT) and active approaches, and how recon feeds into the six-layer enumeration model.

## Overview

Reconnaissance is the initial phase of any penetration test. Its goal is to map the target's internet presence, infrastructure, and services as comprehensively as possible before any direct attack is attempted.

A critical distinction: **OSINT is not enumeration**. OSINT is purely passive — you gather information from third-party sources without touching the target's systems. Enumeration involves active queries directed at the target. Confusing these two leads to premature noise or missed passive data.

> "Our goal is not to get at the systems but to find all the ways to get there."

Jumping straight to exploitation (e.g., brute-forcing SSH as soon as you see port 22) is a common mistake. It causes alerts, can get you blacklisted, and ignores the vastly richer picture that methodical recon builds.

## The Six-Layer Enumeration Model

The full footprinting methodology divides investigation into six nested layers. Recon covers the outermost layers; deeper layers are addressed during service enumeration.

| Layer | Description | Information Categories |
|-------|-------------|----------------------|
| 1. Internet Presence | External infrastructure and online footprint | Domains, subdomains, vHosts, ASN, netblocks, IP addresses, cloud instances, security measures |
| 2. Gateway | Security measures protecting the perimeter | Firewalls, DMZ, IPS/IDS, EDR, proxies, NAC, network segmentation, VPN, Cloudflare |
| 3. Accessible Services | Interfaces and services exposed externally or internally | Service type, functionality, configuration, port, version, interface |
| 4. Processes | Internal processes behind those services | PID, processed data, tasks, source, destination |
| 5. Privileges | Permissions and roles on accessible services | Groups, users, permissions, restrictions, environment |
| 6. OS Setup | Internal system configuration | OS type, patch level, network config, config files, sensitive files |

Recon work is primarily concerned with Layers 1 and 2. Layers 3–6 are the domain of active enumeration.

## Passive vs. Active Recon

**Passive (OSINT)** — No direct contact with the target:
- WHOIS lookups
- Certificate transparency logs (crt.sh)
- Google dorks (`inurl:`, `intext:`, `site:`)
- DNS history via SecurityTrails, ViewDNS
- LinkedIn / job postings for staff and technology enumeration
- Cloud storage discovery via GrayHatWarfare
- Shodan, Censys queries

**Active** — Direct contact with the target:
- DNS queries (dig, host, dnsenum)
- Port scanning (nmap)
- Banner grabbing
- Service-specific scripts and probes

## The Three Core Principles

1. There is more than meets the eye. Consider all points of view.
2. Distinguish between what we see and what we do not see.
3. There are always ways to gain more information. Understand the target.

Ask yourself at every step:
- What can we see?
- What reasons can we have for seeing it?
- What does the picture we see tell us?
- What can we **not** see, and why?

## Recommended Order of Operations

1. Identify all domains, subdomains, and ASN blocks (Layer 1 passive)
2. Identify cloud resources — S3 buckets, Azure blobs, GCP storage
3. Enumerate staff via LinkedIn; map email patterns and technology stack from job posts
4. Identify security posture clues (Cloudflare, WAF, CDN) — Layer 2
5. Active DNS recon — zone transfers, subdomain brute-force
6. Port scan in-scope hosts
7. Service-level enumeration (Layers 3–6)

## Gotchas & Notes

- Brute-forcing is loud. Exhaust passive and low-noise active methods first.
- DNS zone transfers often reveal internal hostnames, IP ranges, and service roles that would otherwise take hours of manual enumeration.
- SSL certificates frequently expose organizational structure: hostnames, email addresses, and sometimes internal subdomain names appear in SANs.
- Cloud storage misconfigurations are a high-value, low-noise finding — check for unauthenticated access before touching anything else.
- Note.md (internal wiki note): Note that the "Internet Presence" layer intentionally omits human/OSINT factors from the formal model for simplicity — in practice, staff enumeration is part of Layer 1.

## Related Pages

- [[recon/osint/domain_information]]
- [[recon/osint/cloud_resources]]
- [[recon/osint/staff_enumeration]]
- [[enumeration/_overview]]
- [[tools/dnsenum]]
- [[tools/dig]]

## Sources

- raw/footprinting/introduction_enumeration_principles.md
- raw/footprinting/introduction_enumeration_methodology.md
