---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# dig

dig (Domain Information Groper) is the standard DNS query tool for manual DNS enumeration — querying specific record types, attempting zone transfers, and fingerprinting DNS servers.

## Overview

dig is the go-to tool for manual DNS queries. It provides complete control over query type, target nameserver, and output format. More detailed output than `host`, more flexible than `nslookup`.

Available on virtually all Linux distributions by default (part of the `bind9-utils` / `bind-utils` package).

## Commands / Syntax

```bash
# Basic A record query (default)
dig inlanefreight.com

# Query specific record types
dig a inlanefreight.htb @10.129.14.128          # A records (IPv4)
dig aaaa inlanefreight.htb @10.129.14.128        # AAAA records (IPv6)
dig ns inlanefreight.htb @10.129.14.128          # Name servers
dig mx inlanefreight.htb @10.129.14.128          # Mail servers
dig txt inlanefreight.htb @10.129.14.128         # TXT records (SPF, DMARC, etc.)
dig soa inlanefreight.htb @10.129.14.128         # SOA (zone admin info)
dig ptr 8.8.8.8.in-addr.arpa                     # Reverse lookup
dig cname www.inlanefreight.htb @10.129.14.128   # CNAME

# Query all record types
dig any inlanefreight.htb @10.129.14.128

# Zone transfer (AXFR)
dig axfr inlanefreight.htb @10.129.14.128
dig axfr internal.inlanefreight.htb @10.129.14.128

# DNS server version fingerprint (BIND)
dig CH TXT version.bind 10.129.120.85

# Subdomain brute-force with bash loop
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do
    dig $sub.inlanefreight.htb @10.129.14.128 \
    | grep -v ';\|SOA' \
    | sed -r '/^\s*$/d' \
    | grep $sub \
    | tee -a subdomains.txt
done

# Short output format
dig +short ns inlanefreight.com

# Answer section only
dig +noall +answer mx inlanefreight.com

# Trace resolution path
dig +trace inlanefreight.com
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `@<server>` | Specify DNS server to query |
| `-t <type>` | Record type (A, AAAA, MX, NS, TXT, SOA, PTR, CNAME, ANY, AXFR) |
| `+short` | Short output (just the answer values) |
| `+noall` | Suppress all output sections |
| `+answer` | Show only the answer section |
| `+trace` | Trace DNS resolution from root |
| `+noquestion` | Suppress question section |
| `+nocomments` | Suppress comments |
| `CH TXT` | Chaos class query (for BIND version) |
| `-p <port>` | Non-standard port |
| `-4` / `-6` | Force IPv4 / IPv6 |
| `+tcp` | Use TCP instead of UDP |

## Reading dig Output

```
; <<>> DiG 9.16.1 <<>> ns inlanefreight.htb @10.129.14.128
;; QUESTION SECTION:
;inlanefreight.htb.    IN    NS

;; ANSWER SECTION:
inlanefreight.htb.    604800    IN    NS    ns.inlanefreight.htb.

;; ADDITIONAL SECTION:
ns.inlanefreight.htb.    604800    IN    A    10.129.34.136

;; Query time: 0 msec
;; SERVER: 10.129.14.128#53(10.129.14.128)
```

- `604800` = TTL in seconds (604800 = 7 days)
- `IN` = Internet class
- `SERVER:` line confirms which DNS server was actually queried

## Gotchas & Notes

- **AXFR attempt**: Always try zone transfer on any DNS server you discover. A successful AXFR dumps the entire zone — far faster than brute-forcing.
- **`@` syntax**: Without `@<server>`, dig queries the system's default resolver. For internal targets, always specify the target DNS server.
- **`ANY` queries**: Some servers restrict ANY queries but still respond to individual record type queries. If ANY fails, query each type individually.
- **SOA email**: The SOA record's email address uses `.` instead of `@`. `awsdns-hostmaster.amazon.com` means `awsdns-hostmaster@amazon.com`.
- **PTR records**: Reverse lookups reveal hostnames for IPs. Use `dig -x <IP>` as shorthand for the `.in-addr.arpa` format.
- **TCP mode** (`+tcp`): Use for zone transfers and large responses that exceed UDP 512-byte limit.
- **`CH TXT version.bind`**: The BIND server must have this record configured. Not always present but worth trying.

## Related Pages

- [[enumeration/dns]]
- [[tools/dnsenum]]
- [[recon/osint/domain_information]]

## Sources

- raw/footprinting/host_based_enumeration_dns.md
