---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# onesixtyone

onesixtyone is a fast SNMP community string brute-force scanner that identifies valid community strings on SNMP-enabled devices.

## Overview

onesixtyone sends SNMP requests with each candidate community string from a wordlist and records which strings receive a valid response. Much faster than snmpwalk for discovery — designed for scanning multiple hosts with many candidate community strings.

Install:
```bash
sudo apt install onesixtyone
```

## Commands / Syntax

```bash
# Brute-force community strings on a single host
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128

# Scan multiple hosts from a file
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt -i hosts.txt

# Save output to file
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128 -o results.txt

# Faster scan with reduced wait time
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128 -w 100
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-c <wordlist>` | Community string wordlist |
| `-i <file>` | File containing target IP addresses |
| `-o <file>` | Output file |
| `-t <seconds>` | Timeout per host |
| `-w <ms>` | Wait time between requests (milliseconds) |
| `-d` | Debug output |

## Recommended Wordlists

| Wordlist | Location |
|----------|----------|
| SNMP community strings | `/opt/useful/seclists/Discovery/SNMP/snmp.txt` |
| SecLists SNMP | `SecLists/Discovery/SNMP/snmp-onesixtyone.txt` |

For targeted environments, generate custom wordlists using patterns derived from discovered hostnames, organization names, or city/location names.

## Gotchas & Notes

- **UDP**: SNMP uses UDP 161. Standard TCP-based tools won't reach it.
- **Custom community strings**: Administrators often use hostnames, location names, or departmental abbreviations. If default wordlists fail, build a targeted list from known organization info.
- onesixtyone is fast but the response contains the system description — capture this: `10.129.14.128 [public] Linux htb 5.11.0-37-generic ...` gives you both the community string and OS details in one line.
- After finding the community string, use [[tools/enumeration/snmpwalk]] or braa for full OID enumeration.

## Installation

```bash
# Check if installed
which onesixtyone 2>/dev/null || echo "not installed"

# Install (Kali / Parrot)
sudo apt install onesixtyone -y

# Verify
onesixtyone 2>&1 | head -3
```

## Related Pages

- [[enumeration/snmp]]
- [[tools/enumeration/snmpwalk]]

## Sources

- raw/footprinting/host_based_enumeration_snmp.md
