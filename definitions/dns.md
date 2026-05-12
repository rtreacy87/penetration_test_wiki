---
tags: [definition, concept, enumeration/dns]
module: core
last_updated: 2026-05-12
source_count: 0
---

# DNS — Concepts for Penetration Testers

A primer on how DNS works, what zone transfers are, what AXFR means, and how to detect whether a nameserver allows unrestricted zone transfers.

---

## What is DNS?

DNS (Domain Name System) is the internet's phonebook. It translates human-readable names like `inlanefreight.htb` into IP addresses that computers route to. When your browser visits `example.com`, a DNS lookup happens first — the name is resolved to an IP, and only then does the connection go out.

Every lookup is a question-and-answer exchange:

```
You ask:  "What is the IP address for inlanefreight.htb?"
DNS says: "10.129.14.128"
```

This question goes to a **nameserver** — a server whose job is to answer DNS queries. There are several layers of nameservers involved in a real lookup, but from a pentesting perspective, the one that matters most is the **authoritative nameserver**: the server that holds the actual DNS records for a domain and is the final authority on what they say.

---

## DNS Records — What Is Actually Stored

A nameserver doesn't just store IPs. It stores a set of **resource records**, each with a type that tells you what kind of information it contains:

| Record type | What it stores | Pentester interest |
|-------------|---------------|--------------------|
| `A` | IPv4 address for a hostname | Maps hostnames to targets |
| `AAAA` | IPv6 address | Same, IPv6 |
| `MX` | Mail server for a domain | Mail infrastructure; useful for SMTP attacks |
| `NS` | Authoritative nameservers for the domain | Tells you which servers to query for AXFR |
| `CNAME` | Alias — points a name to another name | Dangling CNAMEs → subdomain takeover |
| `TXT` | Arbitrary text data | SPF, DKIM, verification tokens, and sometimes flags or credentials |
| `PTR` | Reverse DNS — IP to hostname | Useful during internal network enumeration |
| `SOA` | Start of Authority — metadata about the zone | Zone serial number, primary NS, admin contact |
| `SRV` | Service location — host + port for a service | AD Kerberos, SIP, XMPP discovery |

During enumeration you typically want all of these, not just `A` records.

---

## What Is a DNS Zone?

A **zone** is a slice of the DNS namespace that a single nameserver is responsible for. It is the collection of all records for a domain and the subdomains it manages directly.

Example:
- `inlanefreight.htb` is a zone. The nameserver for this zone holds the `A`, `MX`, `TXT`, and `NS` records for the root domain.
- `hr.inlanefreight.htb` can be its own separate zone, with its own `SOA` record and its own set of records. A different nameserver might be responsible for it, or the same one might manage both.

The key point: **a zone boundary is not always at a dot**. A zone transfer of `inlanefreight.htb` only returns records managed in that specific zone. Records in a child zone like `hr.inlanefreight.htb` are not included — you have to request that zone separately.

---

## What Is a Zone Transfer?

A **zone transfer** is a mechanism for copying all DNS records in a zone from one nameserver to another. It was designed for legitimate operational use: when a domain has multiple nameservers for redundancy, the primary (master) nameserver pushes its full record set to the secondary (slave) nameservers using zone transfer so they stay in sync.

The intended flow:
```
Primary NS ──zone transfer──► Secondary NS 1
           ──zone transfer──► Secondary NS 2
```

The security problem: the zone transfer protocol doesn't distinguish between a legitimate secondary nameserver and a random attacker asking for the same data. If the nameserver isn't configured to restrict which IPs can request a zone transfer, **anyone can request a full copy of all DNS records in the zone.**

From a pentesting perspective, an unrestricted zone transfer is equivalent to the organization handing you a complete internal network map — every hostname, IP, mail server, internal service, and subdomain they have registered.

---

## What Does AXFR Stand For?

**AXFR** stands for **Authoritative Transfer** (or more precisely, it is the DNS query type code for a full zone transfer, defined in RFC 1035 as "a request for a transfer of an entire zone").

There are two zone transfer types:

| Type | Name | What it does |
|------|------|-------------|
| `AXFR` | Full zone transfer | Transfers every record in the zone from scratch |
| `IXFR` | Incremental zone transfer | Transfers only records that changed since a given serial number |

In practice, AXFR is what you use for exploitation. IXFR is an optimization for legitimate secondary nameservers that already have most of the zone and only need the delta.

When you run `dig axfr inlanefreight.htb @10.129.x.x`, you are sending an AXFR query type to that nameserver, asking it to send you the complete zone. A permissive nameserver responds with every record. A restricted one refuses.

---

## How to Tell If Zone Transfers Are Restricted

Run the AXFR and look at what comes back. There are three possible outcomes:

### 1 — Unrestricted: full zone data returned

```bash
dig axfr inlanefreight.htb @10.129.x.x
```

A successful (unrestricted) response looks like:

```
; <<>> DiG 9.18.x <<>> axfr inlanefreight.htb @10.129.x.x
;; global options: +cmd
inlanefreight.htb.      604800  IN  SOA   ns.inlanefreight.htb. ...
inlanefreight.htb.      604800  IN  NS    ns.inlanefreight.htb.
inlanefreight.htb.      604800  IN  A     10.129.x.x
helpdesk.inlanefreight.htb. 604800 IN A  10.129.x.x
hr.inlanefreight.htb.   604800  IN  A     10.129.x.x
...
inlanefreight.htb.      604800  IN  SOA   ns.inlanefreight.htb. ...
```

Key signals: the response starts and ends with an SOA record (that is how AXFR is framed), and you see multiple different hostnames. If you get records like this, the zone is wide open.

### 2 — Restricted: transfer refused

```
; <<>> DiG 9.18.x <<>> axfr inlanefreight.htb @10.129.x.x
;; global options: +cmd
; Transfer failed.
```

or

```
inlanefreight.htb.      3600  IN  SOA  ns1.inlanefreight.htb. ...
;; XFR size: 1 message
;; Query time: 4 msec
```

If you get only the SOA record (one record, no A/MX/TXT/CNAME entries), the server answered but restricted the transfer — it gave you the SOA to confirm the zone exists but nothing else.

If you see `Transfer failed` or `REFUSED`, the nameserver is explicitly rejecting AXFR from your IP.

### 3 — Wrong target: SERVFAIL or NXDOMAIN

```
;; Got SERVFAIL reply from 10.129.x.x
```

or

```
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN
```

This means the server you queried is not authoritative for that zone — it doesn't have the records to transfer. Make sure you're querying the correct nameserver. First look up the NS records:

```bash
dig NS inlanefreight.htb @10.129.x.x
```

Then run AXFR against the NS record it returns.

---

## Quick Testing Checklist

```bash
# 1. Confirm the nameserver and the zone it's authoritative for
dig NS inlanefreight.htb @STMIP

# 2. Attempt AXFR on the root zone
dig axfr inlanefreight.htb @STMIP

# 3. If you find subdomains (from AXFR or brute-force), try AXFR on each
dig axfr hr.inlanefreight.htb @STMIP

# 4. Try all NS records — some restrict AXFR on only the primary
for ns in $(dig +short NS inlanefreight.htb @STMIP); do
    echo "=== $ns ==="
    dig axfr inlanefreight.htb @$ns
done
```

**Always query the authoritative nameserver directly** using `@IP`. If you let the query go through the system resolver, it will contact a public DNS server that knows nothing about internal or lab domains.

---

## Why Zone Transfers Matter for Pentesting

When zone transfers are unrestricted, a single `dig axfr` command gives you:

- Every hostname in the domain — internal systems, admin panels, VPNs, dev environments
- Internal IP addresses (RFC 1918 ranges) that confirm network topology
- Mail server infrastructure (`MX`) for SMTP attacks
- TXT records that may contain credentials, API keys, SPF/DKIM config revealing email providers
- Subdomain inventory that would otherwise require days of brute-forcing

In a well-secured environment, zone transfers are locked down to specific IP ranges (the secondary nameservers only). In a misconfigured one, any host on the internet can request a complete organizational DNS dump.

---

## Where to Go Next

| Goal | Page |
|------|------|
| Enumerate DNS records manually with dig | [[enumeration/dns]] |
| Brute-force subdomains (especially internal/HTB domains) | [[tools/enumeration/subbrute]] |
| Automated NS/MX/AXFR/brute-force in one run | [[tools/enumeration/dnsenum]] |
| DNS query syntax reference | [[tools/enumeration/dig]] |
| Exploit unrestricted AXFR, subdomain takeover, cache poisoning | [[attack/dns]] |
| Lab example: find a flag via subbrute + AXFR | [[labs/htb/attacking_common_services/dns_subdomain_enumeration_and_zone_transfer]] |
| DNS port and protocol basics | [[ports/common_ports]] |

## Related pages

- [[enumeration/dns]]
- [[attack/dns]]
- [[tools/enumeration/dig]]
- [[tools/enumeration/dnsenum]]
- [[tools/enumeration/subbrute]]
- [[definitions/_overview]]
