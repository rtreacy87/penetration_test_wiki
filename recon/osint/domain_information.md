---
tags: [recon, recon/osint]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Domain Information

Passive domain reconnaissance: WHOIS, certificate transparency logs, DNS history, and the tools used to map a target's external domain footprint without making contact.

## Overview

Domain information gathering is the first step of infrastructure-based recon. It reveals the structure of a company's public internet presence — domains, subdomains, IP ranges, name servers, email systems, and cloud dependencies — all without sending a single packet to the target.

The goal is to build a complete map of the target's internet presence that becomes the target list for subsequent active enumeration.

## Key Concepts / Techniques

### WHOIS

WHOIS databases contain registration data for domains: registrar, registrant contact, name servers, and registration/expiry dates. Even with privacy protection masking individual contacts, WHOIS still reveals:
- Registrar identity
- Name server details
- Registration patterns (naming conventions, age of domain)

### Certificate Transparency (crt.sh)

TLS certificates are logged publicly in certificate transparency databases. Searching crt.sh reveals subdomains that have been issued certificates — including internal-facing subdomains that may appear in SANs (Subject Alternative Names).

This is one of the highest-yield passive techniques because it is comprehensive, automated (certificates must be logged), and often reveals development/staging/internal subdomains.

```bash
# Query crt.sh for all certificates related to a domain
curl -s "https://crt.sh/?q=inlanefreight.com&output=json" | jq .
```

### DNS History

Services like SecurityTrails and ViewDNS.info maintain historical DNS records. Historical A records can reveal:
- Previous IP addresses (hosting changes, data center moves)
- Infrastructure that may still be running old software
- Previously disclosed subdomains that no longer appear in current DNS

### DNS Record Analysis

Even without a zone transfer, querying individual record types reveals service architecture:

| Record | What it reveals |
|--------|----------------|
| A/AAAA | IP addresses, hosting providers |
| MX | Email provider (Google Workspace, Microsoft 365, or self-hosted) |
| NS | Name servers (often reveal registrar or hosting) |
| TXT | SPF, DMARC, DKIM (reveals email providers); domain verification tokens (reveals third-party services like Atlassian, Google, etc.) |
| SOA | Primary NS and admin email |
| CNAME | Aliases that reveal third-party service usage |

### Third-Party Intelligence Platforms

| Tool | What it provides |
|------|-----------------|
| Shodan | Internet-wide scan data: open ports, banners, service versions |
| Censys | Similar to Shodan, with strong TLS certificate data |
| SecurityTrails | DNS history, subdomain enumeration, WHOIS |
| ViewDNS.info | Reverse IP lookup, DNS history, WHOIS |
| domain.glass | Infrastructure intelligence including CDN/WAF detection |

## Commands / Syntax

```bash
# WHOIS lookup
whois inlanefreight.com

# Query crt.sh for subdomains (JSON output)
curl -s "https://crt.sh/?q=%.inlanefreight.com&output=json" | jq -r '.[].name_value' | sort -u

# Basic dig queries
dig a inlanefreight.com
dig mx inlanefreight.com
dig ns inlanefreight.com
dig txt inlanefreight.com
dig any inlanefreight.com

# SOA record (reveals admin contact email)
dig soa www.inlanefreight.com

# DNS version query (BIND fingerprint)
dig CH TXT version.bind 10.129.120.85
```

## Flags & Options

| Source | What to look for |
|--------|-----------------|
| WHOIS | NS records, registrar, registration date, registrant org |
| crt.sh | All SANs: `%.domain.com` catches wildcard certs and subdomains |
| MX records | Reveals email provider — self-hosted MX means enumerable mail server |
| TXT records | SPF entries reveal all authorized sending IPs and mail services; verification tokens reveal active third-party integrations |
| NS records | Non-standard nameservers may indicate self-managed DNS worth probing for zone transfers |

## Gotchas & Notes

- SPF TXT records are goldmines: `v=spf1 include:mailgun.org include:_spf.google.com ip4:10.129.124.8` reveals the email provider stack and sometimes internal IP ranges.
- Certificate SANs regularly include internal hostnames, VPN endpoints, and development servers. Always pull the full SAN list.
- The SOA record's email field uses `.` instead of `@` — `awsdns-hostmaster.amazon.com` in the SOA output means the admin email is `awsdns-hostmaster@amazon.com`.
- Domain registration with privacy protection still leaks NS records and SOA. Focus there.
- Historical records on SecurityTrails can reveal infrastructure that predates a company's cloud migration and may still be running.

## Related Pages

- [[recon/_overview]]
- [[recon/osint/cloud_resources]]
- [[recon/osint/staff_enumeration]]
- [[enumeration/dns]]
- [[tools/dig]]
- [[tools/dnsenum]]

## Sources

- raw/footprinting/infastructure_based_enumeration_domain_information.md
