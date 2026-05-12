---
tags: [enumeration, enumeration/dns, protocol]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# DNS Enumeration

DNS enumeration: record types, zone transfer exploitation, subdomain brute-forcing, and key tools (dig, host, dnsenum, fierce).

## Overview

DNS is the directory of the internet. For a pentester, it is the single most information-rich unauthenticated service available. A misconfigured DNS server can expose an organization's entire internal naming structure through a zone transfer. Even without that, querying individual record types and brute-forcing subdomains builds a detailed map of the target.

DNS operates primarily on UDP port 53, with TCP port 53 used for zone transfers and large responses.

## Key Concepts / Techniques

### DNS Server Types

| Server Type | Description |
|-------------|-------------|
| DNS Root Server | Top-level authority; 13 exist globally; managed by ICANN |
| Authoritative Nameserver | Holds the definitive records for a zone |
| Non-authoritative Nameserver | Caches and forwards; data may be stale |
| Caching DNS Server | Stores responses for TTL duration |
| Forwarding Server | Relays queries to another DNS server |
| Resolver | Client-side resolution (OS level) |

### DNS Record Types

| Record | Description |
|--------|-------------|
| A | IPv4 address for a hostname |
| AAAA | IPv6 address for a hostname |
| MX | Mail server for the domain |
| NS | Authoritative name servers for the domain |
| TXT | Free-form text; used for SPF, DMARC, DKIM, and domain ownership verification |
| CNAME | Alias pointing to another hostname |
| PTR | Reverse lookup: IP to hostname |
| SOA | Zone authority and admin contact |

### Zone Transfers (AXFR)

Zone transfers (AXFR = Asynchronous Full Transfer Zone) synchronize DNS data between primary and secondary name servers. If `allow-transfer` is set to `any` or an overly broad subnet, anyone can request a full copy of the zone file.

A successful zone transfer dumps every A, MX, NS, CNAME, and TXT record in the zone — essentially a complete internal host inventory.

### Subdomain Brute-Forcing

When zone transfers fail, brute-force with a wordlist. Each potential subdomain is queried; successful resolutions indicate live hosts.

### Dangerous DNS Misconfigurations

| Setting | Risk |
|---------|------|
| `allow-transfer any` | Anyone can dump the zone |
| `allow-recursion any` | DNS amplification attacks |
| `allow-query any` | Leaks internal records |
| `zone-statistics on` | Information disclosure |

## Commands / Syntax

```bash
# Query NS records — find all name servers
dig ns inlanefreight.htb @10.129.14.128

# Query SOA — find zone admin email
dig soa www.inlanefreight.com

# Query any/all records
dig any inlanefreight.htb @10.129.14.128

# Attempt zone transfer (AXFR)
dig axfr inlanefreight.htb @10.129.14.128

# Zone transfer on a subdomain/internal zone
dig axfr internal.inlanefreight.htb @10.129.14.128

# Enumerate version (BIND fingerprint)
dig CH TXT version.bind 10.129.120.85

# Subdomain brute-force with bash loop
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do
    dig $sub.inlanefreight.htb @10.129.14.128 | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt
done

# dnsenum — comprehensive: NS, MX, zone transfer attempt, brute-force
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 -o subdomains.txt \
    -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
    inlanefreight.htb

# host — simpler alternative to dig
host -t ns inlanefreight.com
host -t mx inlanefreight.com
host -l inlanefreight.htb ns.inlanefreight.htb  # zone transfer attempt
```

## Flags & Options

### dig

| Flag | Description |
|------|-------------|
| `@<server>` | Specify DNS server to query |
| `-t <type>` | Record type (A, MX, NS, TXT, ANY, AXFR) |
| `+short` | Abbreviated output |
| `+noall +answer` | Show only the answer section |
| `CH TXT` | Query chaos class (for version.bind) |

### dnsenum

| Flag | Description |
|------|-------------|
| `--dnsserver <IP>` | Target DNS server |
| `--enum` | Enumerate all record types |
| `-p 0` | No Google scraping |
| `-s 0` | No Scrap subdomains |
| `-o <file>` | Output file |
| `-f <wordlist>` | Wordlist for brute-force |

## Version Detection & Exploit Research

The DNS server software and version are rarely exposed directly, but BIND (the most common authoritative nameserver) supports a special chaos-class TXT record that returns the version string. This is the primary pre-authentication fingerprint. Version information narrows the search space for known BIND vulnerabilities significantly.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| BIND version query | `dig CH TXT version.bind <IP>` | BIND version string (e.g., `9.16.1-Ubuntu`) |
| SOA record | `dig SOA <domain> @<IP>` | Authoritative NS + admin email |
| NSE script | `nmap -p53 --script dns-nsid <IP>` | Nameserver ID, version (if disclosed) |
| nmap service scan | `nmap -sV -p53 <IP>` | Software and version from banner |

### Searching for Exploits

```bash
# Searchsploit — offline ExploitDB search
searchsploit bind 9
searchsploit named

# Metasploit — search for DNS modules
msf6> search type:exploit name:bind
msf6> search type:exploit name:named
```

### Notable CVEs

| CVE | Affected Versions | Impact |
|-----|------------------|--------|
| CVE-2021-25214 | BIND 9.16.x < 9.16.11 | Assertion failure / crash (DoS) |
| CVE-2020-8617 | BIND 9.x < 9.11.19 / 9.16.3 | Assertion failure via TSIG records (DoS) |
| CVE-2020-8616 | BIND 9.x < 9.11.19 | NXNSAttack DNS amplification |
| CVE-2016-2776 | BIND 9.x < 9.9.9-P3 | Crafted query causes crash (DoS) |
| CVE-2008-1447 | All DNS resolvers (Kaminsky) | Cache poisoning via birthday attack |

## Gotchas & Notes

- Zone transfers succeed silently — if the server allows them, dig returns all records with no error. Always attempt AXFR on any open DNS server.
- AXFR on the base zone (e.g., `inlanefreight.htb`) may succeed but miss records in sub-zones (e.g., `internal.inlanefreight.htb`). Try all NS records and all discovered sub-zones.
- Non-authoritative nameservers may have stale or incomplete data. Always query the authoritative NS.
- TXT records are intelligence goldmines: SPF reveals all authorized email infrastructure including internal IP ranges; verification tokens reveal every third-party service the company uses.
- DNS over HTTPS (DoH) and DNS over TLS (DoT) are not typically configured for internal servers — target internal DNS with standard UDP/TCP 53.
- `dig any` does not guarantee all records are returned; the server may restrict what it discloses.

## Related Pages

- [[attack/dns]] — exploitation: zone transfer abuse, subdomain takeover, DNS cache poisoning
- [[recon/osint/domain_information]]
- [[tools/dig]]
- [[tools/dnsenum]]
- [[enumeration/_overview]]

## Sources

- raw/footprinting/host_based_enumeration_dns.md
