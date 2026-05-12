---
tags: [tool, enumeration/dns]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# subbrute

DNS subdomain brute-forcer with explicit resolver support; the go-to tool when you need to enumerate subdomains against a specific nameserver rather than the system resolver.

## Overview

subbrute sends DNS queries for candidate subdomain names and reports which ones resolve. Its defining feature is the `-r resolvers.txt` flag, which lets you specify exactly which DNS server receives those queries. This makes it the right tool whenever the target domain is not in public DNS — internal lab domains (`.htb`, `.local`), air-gapped environments, and any scenario where the authoritative nameserver is not discoverable through the normal resolution chain.

subbrute is not on Kali by default and must be cloned from GitHub. It is a lightweight Python script, not an installed binary.

**Use subbrute when:**
- The domain only exists on a target-controlled nameserver (HTB labs, internal engagements)
- You need to aim brute-force at one specific nameserver, not the system resolver or any public DNS
- You want a simple, fast subdomain wordlist sweep with minimal dependencies

**Use dnsenum or subfinder instead when:**
- The domain is in public DNS and you want passive sources (certificate transparency, Bing) combined with active brute-force in one run — dnsenum does this automatically
- You need MX, NS, and AXFR in the same run — dnsenum covers all of these; subbrute only brute-forces names
- You want multi-source aggregation across dozens of passive APIs — use subfinder

See the comparison table below.

## Installation

subbrute is not packaged for apt. Clone it directly:

```bash
# Clone
git clone https://github.com/TheRook/subbrute.git && cd subbrute/

# Verify
python3 subbrute.py --help
```

No additional Python packages are required beyond the standard library.

---

## Usage

### Basic — enumerate against the system resolver

```bash
python3 subbrute.py example.com \
    -s /opt/useful/SecLists/Discovery/DNS/namelist.txt
```

This works for public domains but will return nothing for internal/HTB domains that don't exist in public DNS.

### Target a specific nameserver (most common use case)

```bash
# Write the target IP into resolvers.txt
echo 10.129.x.x > resolvers.txt

python3 subbrute.py inlanefreight.htb \
    -s /opt/useful/SecLists/Discovery/DNS/namelist.txt \
    -r resolvers.txt
```

All DNS queries go to `10.129.x.x` instead of the system resolver. Required for `.htb` and other internal domains.

### Pipe output into a zone transfer loop

After subbrute identifies subdomains, transfer each zone immediately:

```bash
python3 subbrute.py inlanefreight.htb \
    -s /opt/useful/SecLists/Discovery/DNS/namelist.txt \
    -r resolvers.txt 2>/dev/null | while read sub; do
    echo "=== $sub ==="
    dig axfr "$sub" @10.129.x.x | grep -v "^;" | grep -v "^$"
done
```

---

## Flags & Options

| Flag | Description |
|------|-------------|
| (positional) | Target domain to enumerate under (e.g., `example.com`) |
| `-s <wordlist>` | Path to subdomain name wordlist. `/opt/useful/SecLists/Discovery/DNS/namelist.txt` is a good default. |
| `-r <resolvers>` | File containing one or more IP addresses to use as DNS resolvers. One IP per line. Overrides the system resolver entirely. |
| `-p <processes>` | Number of parallel processes (default: 16). Reduce if the target rate-limits. Increase for faster results on a stable connection. |
| `-o <outfile>` | Write discovered subdomains to a file in addition to stdout. |
| `-t <type>` | DNS record type to query (default: `ANY`). Rarely needs changing. |

**Resolver count warning:** subbrute prints `Warning: Fewer than 16 resolvers per process` if `resolvers.txt` has fewer than 16 entries. This is cosmetic — one resolver works fine for a lab target. In production engagements, adding multiple resolvers speeds up enumeration by distributing queries.

---

## subbrute vs dnsenum vs subfinder

| | subbrute | dnsenum | subfinder |
|---|---|---|---|
| **Primary use** | Wordlist brute-force against a chosen nameserver | Brute-force + NS/MX/AXFR in one run | Passive multi-source aggregation |
| **Custom resolver (`-r`)** | Yes — core feature | Via `--dnsserver` flag | No |
| **Works on internal/HTB domains** | Yes | Yes (with `--dnsserver`) | No — needs public passive sources |
| **Passive sources (crt.sh, etc.)** | No | Limited | Yes — 40+ sources |
| **Zone transfer (AXFR)** | No — pipe to `dig` separately | Yes — built in | No |
| **MX / NS enumeration** | No | Yes | No |
| **Speed** | Fast | Moderate | Fast (passive) |
| **Installation** | `git clone` | `sudo apt install dnsenum` | `go install` or apt |

**Decision rule:** for internal or HTB domains where public passive sources are useless, start with subbrute (custom resolver) and follow up with `dig axfr` on each discovered subdomain. For public domains, start with subfinder for passive coverage, then run dnsenum for active brute-force and zone transfer in one pass.

---

## Gotchas & Notes

- The `-r resolvers.txt` flag is what separates subbrute from other tools — without it, subbrute uses the system resolver and will return nothing for `.htb` domains
- subbrute does not perform zone transfers — it only identifies which names resolve; run `dig axfr` separately for each discovered subdomain
- The resolver count warning is harmless in lab contexts; one resolver is enough
- Use `2>/dev/null` to suppress the warning when piping output to downstream commands
- subbrute has not been actively maintained since ~2014, but the core brute-force logic is simple and still works reliably

## Related Pages

- [[enumeration/dns]] — DNS record types, AXFR, brute-forcing methodology
- [[attack/dns]] — exploiting AXFR, subdomain takeover, cache poisoning
- [[tools/enumeration/dnsenum]] — all-in-one DNS enumeration (NS, MX, AXFR, brute-force)
- [[tools/enumeration/dig]] — AXFR syntax and record-type queries
- [[labs/htb/attacking_common_services/dns_subdomain_enumeration_and_zone_transfer]] — lab using subbrute against an internal nameserver

## Sources

- raw/lab/attacking_common_services/attacking_dns.md
