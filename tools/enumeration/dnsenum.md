---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# dnsenum

dnsenum is a comprehensive DNS enumeration tool that discovers subdomains, attempts zone transfers, and brute-forces hostnames against a target domain.

## Overview

dnsenum performs multiple DNS operations in a single run: NS record lookup, MX record lookup, zone transfer attempts (AXFR), and subdomain brute-forcing from a wordlist. It consolidates what would otherwise require multiple dig/host commands.

Install:
```bash
sudo apt install dnsenum
```

## Commands / Syntax

```bash
# Basic enumeration against a target domain
dnsenum inlanefreight.com

# Specify a custom DNS server and brute-force subdomains
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 \
    -o subdomains.txt \
    -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
    inlanefreight.htb

# No Google scraping, no Scrap, output to file
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o output.txt \
    -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
    target.htb

# Standard internet-facing target
dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `--dnsserver <IP>` | DNS server to query |
| `--enum` | Shorthand for NS lookup + zone transfer + brute-force |
| `-p <N>` | Number of Google pages to scrape (0 = disable) |
| `-s <N>` | Maximum of subdomains to scrap (0 = disable) |
| `-o <file>` | Write results to file |
| `-f <wordlist>` | Wordlist for brute-force |
| `-r` | Recursive subdomain brute-force on found subdomains |
| `--threads <N>` | Number of threads (default 5) |
| `--timeout <N>` | Timeout per query in seconds |
| `-v` | Verbose |

## What It Discovers

1. **NS records** — Authoritative name servers
2. **MX records** — Mail servers
3. **Zone transfer** (AXFR) — Attempts on each NS found
4. **Subdomain brute-force** — From wordlist against the target domain
5. **IP-to-hostname mapping** — For discovered IPs
6. **Bind version** — If the DNS server discloses it

## Gotchas & Notes

- dnsenum attempts zone transfers against all discovered NS records. If AXFR is allowed, you get the full zone without needing to brute-force.
- The `-r` flag recursively brute-forces discovered subdomains — useful for multi-level structures like `dev.internal.domain.com`.
- Google scraping (`-p`) is noisy and slow; disable it with `-p 0` for internal targets.
- For internal DNS servers, always specify `--dnsserver <IP>` — the system's default resolver won't resolve internal `.htb` zones.
- Output with `-o` writes a clean subdomain list; useful as input for subsequent tools.
- dnsenum is single-threaded by default (5 threads with `--threads`). For large wordlists, consider using the bash loop approach with dig for more control.

## Installation

```bash
# Check if installed
dnsenum --version 2>/dev/null || dnsenum -h 2>/dev/null | head -1 || echo "not installed"

# Install (Kali / Parrot)
sudo apt install dnsenum -y

# Verify
dnsenum --help 2>&1 | head -3
```

## Related Pages

- [[enumeration/dns]]
- [[tools/enumeration/dig]]
- [[recon/osint/domain_information]]

## Sources

- raw/footprinting/host_based_enumeration_dns.md
