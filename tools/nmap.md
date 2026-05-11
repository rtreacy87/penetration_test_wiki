---
tags: [tool, enumeration]
module: network_enumeration_with_nmap
last_updated: 2026-05-10
source_count: 9
---

# Nmap

Comprehensive reference for Nmap (Network Mapper): host discovery, port and service scanning, OS detection, the Nmap Scripting Engine (NSE), performance tuning, output formats, and firewall/IDS evasion.

## Overview

Nmap is an open-source network analysis and security auditing tool written in C, C++, Python, and Lua. It uses raw packets to identify live hosts, open ports, running services with versions, operating systems, and firewall/IDS configurations. It is one of the most foundational tools in offensive security and is used from initial recon through vulnerability assessment.

**Core scanning pipeline:**

1. Host discovery — is the target alive?
2. Port scanning — which ports are open/closed/filtered?
3. Service enumeration — what is running on each open port, and what version?
4. OS detection — what operating system is the target running?
5. NSE scripting — automated interaction, vulnerability checks, brute-forcing

---

## Syntax

```bash
sudo nmap <scan types> <options> <target>
```

Targets can be specified as:
- Single IP: `10.129.2.28`
- CIDR range: `10.129.2.0/24`
- IP range: `10.129.2.18-20`
- IP list file: `-iL hosts.lst`
- Multiple IPs: `10.129.2.18 10.129.2.19 10.129.2.20`

---

## Host Discovery

Nmap performs host discovery before port scanning. On local networks it defaults to ARP pings; across routed networks it uses ICMP echo requests.

```bash
# Ping sweep a network range (no port scan), save all formats
sudo nmap 10.129.2.0/24 -sn -oA tnet

# Ping sweep from a host list
sudo nmap -sn -oA tnet -iL hosts.lst

# Single host — show why host is marked up
sudo nmap 10.129.2.18 -sn -oA host -PE --reason

# Force ICMP echo requests (disable ARP ping)
sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping

# Show all packets sent and received
sudo nmap 10.129.2.18 -sn --packet-trace
```

**Key host discovery options:**

| Option | Description |
|--------|-------------|
| `-sn` | Disable port scan (ping sweep only) |
| `-PE` | Use ICMP echo requests |
| `-PP` | Use ICMP timestamp requests |
| `-PM` | Use ICMP netmask requests |
| `-PS <ports>` | TCP SYN ping to specified ports |
| `-PA <ports>` | TCP ACK ping to specified ports |
| `-PU <ports>` | UDP ping to specified ports |
| `--disable-arp-ping` | Disable ARP-based host discovery |
| `--reason` | Show reason for host/port state |
| `--packet-trace` | Show all packets sent and received |
| `-iL <file>` | Read targets from a list file |

**Note:** By default on local networks, Nmap sends ARP requests before ICMP. ARP pings often succeed even when ICMP is blocked. Use `--disable-arp-ping` combined with `-PE` to force ICMP-only discovery.

---

## Port States

| State | Description |
|-------|-------------|
| `open` | Connection established (TCP, UDP, or SCTP) |
| `closed` | Port accessible, but no application listening; TCP RST returned |
| `filtered` | No response or error response — firewall likely dropping packets |
| `unfiltered` | Port accessible but open/closed cannot be determined (ACK scan only) |
| `open\|filtered` | No response received; firewall or filter may be present |
| `closed\|filtered` | Only occurs in IP ID idle scans |

---

## Scan Types

### TCP Scan Comparison

| Scan             | Flag           | Mechanism                               | Stealth   | Notes                                                       |
| ---------------- | -------------- | --------------------------------------- | --------- | ----------------------------------------------------------- |
| SCTP (half-open) | `-sS`          | Sends SYN, reads SYN-ACK/RST, sends RST | High      | Default with root; fastest; does not complete handshake     |
| Connect          | `-sT`          | Full three-way handshake                | Low       | Default without root; logged by services; most accurate     |
| ACK              | `-sA`          | Sends ACK only; reads RST               | High      | Does not determine open/closed; useful for firewall mapping |
| Window           | `-sW`          | ACK scan + analyses TCP window field    | Medium    | May reveal open vs. closed in some OSes                     |
| Maimon           | `-sM`          | FIN/ACK probe                           | Medium    | Some BSD systems respond unexpectedly                       |
| Null             | `-sN`          | No flags set                            | High      | Closed ports return RST; open ports drop packet             |
| FIN              | `-sF`          | FIN flag only                           | High      | Same logic as Null scan                                     |
| Xmas             | `-sX`          | FIN, PSH, URG flags                     | High      | Same logic as Null scan                                     |
| Idle             | `-sI <zombie>` | Uses zombie host IP ID sequence         | Very high | Requires a suitable idle host                               |

### Other Scan Types

| Scan | Flag | Notes |
|------|------|-------|
| UDP | `-sU` | Stateless; very slow; ICMP unreachable = closed; no response = open\|filtered |
| IP Protocol | `-sO` | Enumerate supported IP protocols |
| FTP bounce | `-b <host>` | Use FTP relay to scan |
| SCTP INIT | `-sY` | SCTP handshake equivalent of SYN |
| SCTP COOKIE-ECHO | `-sZ` | Harder to detect than INIT |

---

## Port Specification

| Option | Description |
|--------|-------------|
| `-p 22,80,443` | Scan specific ports |
| `-p 22-445` | Scan a port range |
| `-p-` | Scan all 65535 ports |
| `-F` | Fast scan (top 100 ports) |
| `--top-ports=<N>` | Scan the N most common ports |
| `-p U:53,T:22` | Specify UDP or TCP protocol per port |

---

## Service and Version Detection

```bash
# Service version detection on all ports
sudo nmap 10.129.2.28 -p- -sV

# With verbose output to see open ports as discovered
sudo nmap 10.129.2.28 -p- -sV -v

# Show scan progress every 5 seconds
sudo nmap 10.129.2.28 -p- -sV --stats-every=5s

# Version scan with full packet trace (clean, no noise)
sudo nmap 10.129.2.28 -p 445 -Pn -n --disable-arp-ping --packet-trace -sV --reason
```

**Press `[Space Bar]` during a scan to print current progress.**

Nmap first reads service banners, then falls back to signature-based matching if the banner is ambiguous. Banner grabbing manually with `nc` or `tcpdump` can reveal additional information that Nmap's automated matching misses (e.g., OS distribution strings inside SMTP/FTP banners).

```bash
# Manual banner grab for comparison
nc -nv 10.129.2.28 25
```

---

## OS Detection

```bash
# OS detection only
sudo nmap 10.129.2.28 -O

# Aggressive scan: -sV + -O + --traceroute + -sC
sudo nmap 10.129.2.28 -p 80 -A
```

OS detection requires at least one open and one closed port for reliable results. The `-A` flag combines service detection, OS detection, traceroute, and default NSE scripts in a single command — useful for thorough single-target enumeration.

---

## Nmap Scripting Engine (NSE)

NSE scripts are written in Lua and provide automated interaction with services, vulnerability checks, brute-forcing, and more. Scripts live in `/usr/share/nmap/scripts/`.

### Script Categories

| Category | Description |
|----------|-------------|
| `auth` | Determine or test authentication credentials |
| `broadcast` | Host discovery via broadcast (auto-adds results) |
| `brute` | Credential brute-force against services |
| `default` | Executed with `-sC`; safe and widely useful |
| `discovery` | Enumerate accessible services and resources |
| `dos` | Check for denial-of-service vulnerabilities (rarely used) |
| `exploit` | Attempt exploitation of known vulnerabilities |
| `external` | Use external services for processing |
| `fuzzer` | Send malformed/unexpected data to find bugs |
| `intrusive` | May negatively affect target systems |
| `malware` | Detect malware on target |
| `safe` | Non-intrusive, non-destructive scripts |
| `version` | Extension for service/version detection |
| `vuln` | Identify specific known vulnerabilities |

### NSE Usage

```bash
# Run default scripts (-sC is equivalent to --script=default)
sudo nmap <target> -sC

# Run all scripts in a category
sudo nmap <target> --script <category>
sudo nmap 10.129.2.28 -p 80 -sV --script vuln

# Run specific named scripts
sudo nmap <target> --script <script-name>,<script-name>
sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands

# Aggressive scan (sV + O + traceroute + sC)
sudo nmap 10.129.2.28 -p 80 -A
```

### Practical NSE Examples

```bash
# SMTP: grab banner + enumerate supported commands
sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands

# HTTP: enumerate WordPress, check stored XSS, find users
sudo nmap 10.129.2.28 -p 80 -sV --script vuln

# SMB: enumerate shares, users, OS info
sudo nmap 10.129.2.28 -p 445 --script smb-enum-shares,smb-enum-users

# SSH: check supported authentication methods
sudo nmap 10.129.2.28 -p 22 --script ssh-auth-methods

# FTP: check for anonymous login
sudo nmap 10.129.2.28 -p 21 --script ftp-anon

# DNS: enumerate zone transfer possibility
sudo nmap 10.129.2.28 -p 53 --script dns-zone-transfer
```

**Find scripts for a specific protocol:**

```bash
ls /usr/share/nmap/scripts/ | grep smb
ls /usr/share/nmap/scripts/ | grep http
```

---

## Performance Tuning

### Timing Templates

| Template | Alias | Use Case |
|----------|-------|----------|
| `-T0` | paranoid | IDS evasion; 5-min delay between probes |
| `-T1` | sneaky | Slow; evades most rate-based IDS |
| `-T2` | polite | Slower to reduce bandwidth/CPU impact |
| `-T3` | normal | Default; balanced |
| `-T4` | aggressive | Fast; assumes reliable network |
| `-T5` | insane | Fastest; may miss ports on slow networks |

```bash
# Use insane timing (careful — may trigger IDS and miss ports)
sudo nmap 10.129.2.0/24 -F -T5

# Paranoid timing for IDS evasion
sudo nmap 10.129.2.28 -p- -T0
```

### Fine-Grained Performance Options

| Option | Description |
|--------|-------------|
| `--min-rate <N>` | Send at least N packets per second |
| `--max-rate <N>` | Send at most N packets per second |
| `--min-parallelism <N>` | Minimum number of parallel probes |
| `--max-parallelism <N>` | Maximum number of parallel probes |
| `--initial-rtt-timeout <time>` | Initial RTT timeout (e.g., `50ms`) |
| `--max-rtt-timeout <time>` | Maximum RTT timeout (e.g., `100ms`) |
| `--max-retries <N>` | Max retransmissions per port (default 10) |
| `--host-timeout <time>` | Give up on host after this long |
| `--scan-delay <time>` | Minimum delay between probes to a host |

```bash
# Fast network scan with aggressive RTT settings
sudo nmap 10.129.2.0/24 -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms

# Minimize retries to speed up (may miss ports on lossy links)
sudo nmap 10.129.2.0/24 -F --max-retries 0

# High throughput scan (useful in white-box assessments)
sudo nmap 10.129.2.0/24 -F --min-rate 300
```

**Warning:** Aggressive RTT settings and zero retries can cause missed hosts and ports. Always balance speed against completeness, especially in black-box assessments.

---

## Output Formats

| Format | Flag | Extension | Best For |
|--------|------|-----------|----------|
| Normal | `-oN` | `.nmap` | Human reading, quick review |
| Grepable | `-oG` | `.gnmap` | `grep`, `awk`, scripting |
| XML | `-oX` | `.xml` | Parsing, importing into other tools |
| All three | `-oA` | `.nmap` + `.gnmap` + `.xml` | Standard practice — always use this |

```bash
# Save in all formats (recommended for every scan)
sudo nmap 10.129.2.28 -p- -oA target

# Convert XML output to HTML report
xsltproc target.xml -o target.html

# Extract open TCP ports from grepable output
grep "/tcp" target.gnmap | grep "open"

# Extract hosts from grepable output
grep "Status: Up" target.gnmap | cut -d' ' -f2
```

**Always save scan results.** Different tools produce different results, and saved scans allow comparison, documentation, and reporting.

---

## Useful Combined Workflows

### Initial Network Sweep

```bash
# 1. Discover live hosts on a subnet
sudo nmap 10.129.2.0/24 -sn -oA sweep

# 2. Fast scan of live hosts (top 100 ports)
sudo nmap -iL sweep.gnmap -F -oA fast_scan

# 3. Full port scan of interesting hosts
sudo nmap 10.129.2.28 -p- -sV -oA full_scan
```

### Targeted Service Enumeration

```bash
# Quick recon: version detection + default scripts
sudo nmap 10.129.2.28 -sV -sC -oA recon

# Deep scan: all ports + versions + aggressive detection
sudo nmap 10.129.2.28 -p- -sV -A -oA deep_scan

# UDP scan (slower — run on focused port set)
sudo nmap 10.129.2.28 -sU -p 53,67,68,69,123,161,162 -oA udp_scan
```

### Stealth / Evasion Scans

See [[attack/network/firewall_evasion]] for detailed evasion techniques.

```bash
# ACK scan to map firewall rules
sudo nmap 10.129.2.28 -p 21,22,25,80,443 -sA -Pn -n --disable-arp-ping

# Decoy scan (hide real IP among random decoys)
sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping -D RND:5

# Scan from spoofed source IP
sudo nmap 10.129.2.28 -p 445 -S 10.129.2.200 -e tun0 -Pn -n

# Use DNS source port to bypass firewall rules
sudo nmap 10.129.2.28 -p 50000 -sS -Pn -n --disable-arp-ping --source-port 53
```

---

## Flags & Options Quick Reference

| Option | Description |
|--------|-------------|
| `-sS` | TCP SYN scan (default with root) |
| `-sT` | TCP Connect scan (default without root) |
| `-sA` | TCP ACK scan |
| `-sU` | UDP scan |
| `-sV` | Service/version detection |
| `-sC` | Default NSE scripts |
| `-O` | OS detection |
| `-A` | Aggressive: `-sV -O --traceroute -sC` |
| `-p <ports>` | Port specification |
| `-p-` | All ports |
| `-F` | Fast: top 100 ports |
| `--top-ports=N` | Top N ports |
| `-Pn` | Skip host discovery (treat all as up) |
| `-n` | Disable DNS resolution |
| `-v` / `-vv` | Increase verbosity |
| `--packet-trace` | Show all packets |
| `--reason` | Show reason for state |
| `--stats-every=Ns` | Print progress every N seconds |
| `-D RND:N` | Decoy scan with N random IPs |
| `-S <IP>` | Spoof source IP |
| `-e <iface>` | Use specified network interface |
| `--source-port <N>` | Use specified source port |
| `--dns-server <ns>` | Use custom DNS server |
| `--disable-arp-ping` | Disable ARP host discovery |
| `-T <0-5>` | Timing template |
| `--min-rate <N>` | Minimum packet rate |
| `--max-retries <N>` | Maximum port retransmissions |
| `-oN` / `-oG` / `-oX` / `-oA` | Output formats |

---

## Gotchas & Notes

- **Root is required** for SYN scans (`-sS`), OS detection (`-O`), and most evasion techniques. Without root, Nmap defaults to the slower Connect scan (`-sT`).
- **ARP before ICMP:** On local subnets, Nmap sends ARP requests before ICMP echo. Use `--disable-arp-ping` with `-PE` to force ICMP-only discovery.
- **Firewall timeout vs. drop:** A filtered port that causes Nmap to retry 10 times (~2 seconds per port) is being dropped by the firewall. A filtered port that returns an ICMP error immediately is being rejected. Different defensive posture.
- **Banner grabbing limits:** Nmap's `-sV` reads the initial banner and matches signatures. It may miss additional data that services only reveal after protocol negotiation. Supplement with `nc` and `tcpdump`.
- **UDP scan accuracy:** UDP ports report `open|filtered` unless the application sends a response. Consider targeting specific known UDP services rather than broad sweeps.
- **Aggressive timing backfire:** `-T4`/`-T5` and `--min-rate` can trigger rate-based IPS blocks and cause Nmap to miss open ports due to packet loss. Use `-T2` or `-T3` for evasion-sensitive assessments.
- **`-Pn` vs. unreachable hosts:** `-Pn` tells Nmap to skip host discovery and treat targets as up. Useful when ICMP is blocked, but will waste time on genuinely dead hosts.
- **Decoy addresses must be live:** If decoy IPs (`-D`) are not actually online, the target may trigger SYN flood protection and become unreachable.
- **`--max-retries 0`** speeds up scans but risks missing open ports on flaky links.

---

## Related Pages

- [[attack/network/firewall_evasion]]
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
- [[tools/netcat]]

## Sources

- raw/network_enumeration_with_nmap/introduction_enumeration.md
- raw/network_enumeration_with_nmap/introduction_to_nmap.md
- raw/network_enumeration_with_nmap/host_enumeration_host_discovery.md
- raw/network_enumeration_with_nmap/host_enumeration_host_and_port_scanning.md
- raw/network_enumeration_with_nmap/host_enumeration_service_enumeration.md
- raw/network_enumeration_with_nmap/host_enumeration_nmap_scripting_engine.md
- raw/network_enumeration_with_nmap/host_enumeration_performance.md
- raw/network_enumeration_with_nmap/host_enumeration_saving_the_result.md
- raw/network_enumeration_with_nmap/bypass_security_measures_firewall_evasion.md
