---
tags: [lab, attack/network, enumeration/dns]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — DNS Subdomain Enumeration and Zone Transfer

**Difficulty:** Easy
**Target domain:** inlanefreight.htb
**Target IP:** STMIP (your spawned machine IP)

One question: find a flag hidden in a DNS TXT record by brute-forcing subdomains and performing a zone transfer against discovered zones.

---

## Question 1

> Find all available DNS records for the "inlanefreight.htb" domain on the target name server and submit the flag found as a DNS record as the answer.

### Why this works

The lab target is running a name server that is authoritative for `inlanefreight.htb` and its subdomains. Two misconfigurations make the flag accessible:

1. **Zone transfer (AXFR) is unrestricted** — the nameserver will hand over all DNS records for any zone you ask about, including internal subdomains it has authority over. See [[definitions/dns]] for a full explanation of what AXFR is and how to detect whether a nameserver restricts it.
2. **Subdomains are not publicly advertised** — you can't find them through passive sources (crt.sh, Shodan). You have to brute-force them by asking the target nameserver directly whether each candidate name exists.

The flag is stored in a TXT record inside one of the non-root subdomains. A zone transfer on `inlanefreight.htb` itself won't reveal it — you need to enumerate the subdomains first, then transfer each one.

### Step 1 — Clone subbrute

subbrute is a DNS brute-forcer that queries a resolver of your choice rather than the system resolver.

```bash
git clone https://github.com/TheRook/subbrute.git && cd subbrute/
```

### Step 2 — Make the target nameserver reachable

`inlanefreight.htb` doesn't exist in public DNS. Before you can brute-force subdomains or run zone transfers, you need to tell your tools where to send DNS queries. There are two ways to do this:

#### Option A — resolvers.txt (subbrute-native, recommended for this lab)

subbrute accepts a resolver file via `-r`. Writing the target IP into `resolvers.txt` routes all subbrute queries directly to the lab nameserver:

```bash
echo STMIP > resolvers.txt
```

For `dig` zone transfers you still add `@STMIP` explicitly on each command.

**Best when:** you only need DNS tools (subbrute, dig). It is targeted and leaves the system resolver untouched, which avoids polluting name resolution for other tools running in the same session.

#### Option B — /etc/hosts modification (system-wide)

Adding the target to `/etc/hosts` makes `inlanefreight.htb` resolvable by every tool that goes through the OS name-resolution stack:

```bash
echo "STMIP inlanefreight.htb" | sudo tee -a /etc/hosts
```

Tools that benefit from this: `curl`, `wget`, `nmap inlanefreight.htb`, any browser-based test, and anything else that calls `getaddrinfo()` under the hood.

**Limitation:** `/etc/hosts` maps a single hostname to an IP — it does not configure STMIP as a DNS nameserver. subbrute still needs `-r resolvers.txt` to brute-force subdomains, and `dig axfr` still needs `@STMIP` to talk to the right nameserver. This approach extends what you can reach, not how DNS queries are routed.

**Best when:** your workflow mixes DNS tools with web or network tools that need to resolve the hostname naturally. Use it alongside Option A rather than instead of it.

#### Comparison

| | resolvers.txt (`-r`) | /etc/hosts |
|---|---|---|
| **Scope** | subbrute queries only | All tools using the OS resolver |
| **Affects `dig axfr`** | No — still need `@STMIP` | No — still need `@STMIP` |
| **Affects subbrute subdomain brute-force** | Yes — routes queries to target NS | No — subbrute still needs `-r` |
| **Affects curl / browser / nmap** | No | Yes |
| **Requires sudo** | No | Yes (writing to `/etc/hosts`) |
| **Cleanup needed** | Delete `resolvers.txt` | Remove the added line from `/etc/hosts` |
| **Risk of side effects** | None | Can shadow a legitimate hostname if the same name exists elsewhere |

For a pure DNS enumeration lab like this one, `-r resolvers.txt` is sufficient. Add `/etc/hosts` if a later phase requires accessing a web service on the target by hostname.

### Step 3 — Brute-force subdomains

```bash
python3 subbrute.py inlanefreight.htb \
    -s /opt/useful/SecLists/Discovery/DNS/namelist.txt \
    -r resolvers.txt
```

| Part | What it does |
|------|-------------|
| `inlanefreight.htb` | Root domain to brute-force under |
| `-s namelist.txt` | Wordlist of candidate subdomain names to test |
| `-r resolvers.txt` | Use the target nameserver as the resolver — required because the domain only exists locally |

Expected output:

```
Warning: Fewer than 16 resolvers per process, consider adding more nameservers to resolvers.txt.
inlanefreight.htb
helpdesk.inlanefreight.htb
hr.inlanefreight.htb
ns.inlanefreight.htb
```

You now have three subdomains: `helpdesk`, `hr`, and `ns`. The root domain also confirms the nameserver is responding.

### Step 4 — Zone transfer each subdomain

The flag is in a TXT record inside one of these subdomains. Request a full AXFR (zone transfer) for each and grep for TXT records:

```bash
dig axfr inlanefreight.htb @STMIP | grep "TXT"
dig axfr helpdesk.inlanefreight.htb @STMIP | grep "TXT"
dig axfr hr.inlanefreight.htb @STMIP | grep "TXT"
dig axfr ns.inlanefreight.htb @STMIP | grep "TXT"
```

The `@STMIP` flag directs the zone transfer request to the lab nameserver instead of looking up an NS record in public DNS (which would fail for `.htb` domains).

The `hr.inlanefreight.htb` zone transfer returns:

```
hr.inlanefreight.htb.   604800  IN  TXT  "HTB{...}"
```

**Submit the value inside the quotes as the flag.**

---

## Why zone transfer on the subdomain, not the root?

Each subdomain can be its own DNS zone with its own SOA record. A zone transfer of `inlanefreight.htb` only returns records that are authoritative for that zone — not records that are in a delegated child zone like `hr.inlanefreight.htb`. The flag is stored in the `hr` zone, so you must transfer `hr.inlanefreight.htb` specifically to see it.

Think of it like a directory: AXFR of the parent gives you the top-level entries, but you have to open each sub-directory separately.

---

## Alternative: transfer all zones in a loop

To avoid running `dig axfr` manually for each subdomain, pipe subbrute output directly to a loop:

```bash
python3 subbrute.py inlanefreight.htb \
    -s /opt/useful/SecLists/Discovery/DNS/namelist.txt \
    -r resolvers.txt 2>/dev/null | while read sub; do
    echo "=== $sub ==="
    dig axfr "$sub" @STMIP | grep -v "^;" | grep -v "^$"
done
```

This transfers every discovered zone and prints non-comment, non-empty lines. Scan the output for `TXT` records.

---

## Lessons learned

- **Subbrute requires a local resolver** — public DNS knows nothing about `.htb` lab domains; you must point it at the target nameserver with `-r`.
- **Enumerate then transfer** — a zone transfer on the root domain won't expose records in child zones. Find the subdomains first, then transfer each one.
- **TXT records carry arbitrary data** — flags, internal credentials, API keys, and debugging info are commonly hidden here. Always include TXT in zone transfer output when reviewing findings.
- **AXFR on every NS** — some nameservers allow AXFR only on specific servers. Try the AXFR against each NS record if the first attempt fails.

## Related pages

- [[attack/dns]] — full DNS attack reference: AXFR, subdomain takeover, cache poisoning
- [[enumeration/dns]] — passive DNS recon and record-type enumeration
- [[tools/enumeration/dig]] — AXFR syntax, record types, zone transfer options
- [[tools/enumeration/dnsenum]] — automated NS/MX/AXFR/brute-force enumeration in one run
- [[tools/enumeration/subbrute]] — DNS subdomain brute-forcer with custom resolver support

## Sources

- raw/lab/attacking_common_services/attacking_dns.md
