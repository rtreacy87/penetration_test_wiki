---
tags: [enumeration, enumeration/snmp]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# SNMP Enumeration

SNMP enumeration: community string discovery, MIB/OID walking, onesixtyone brute-force, snmpwalk, and braa.

## Overview

SNMP (Simple Network Management Protocol) manages and monitors network devices — routers, switches, servers, IoT devices. It runs on **UDP port 161** for queries and **UDP port 162** for traps (unsolicited notifications from device to manager).

From a pentester's perspective, SNMP is valuable because:
- Versions 1 and 2c transmit everything in plaintext (including community strings)
- A known community string unlocks system-level information: OS version, installed packages, running processes, network interfaces, users, and more
- Default community strings (`public`, `private`) are frequently left unchanged

## Key Concepts / Techniques

### SNMP Versions

| Version | Security |
|---------|----------|
| SNMPv1 | No authentication, no encryption — anyone can read/write |
| SNMPv2c | Community string as password (plaintext); `v2c` = community-based |
| SNMPv3 | Username/password authentication + encryption; complex to configure |

Many organizations run SNMPv2c (or SNMPv1) because SNMPv3 migration is complex. This is the attack surface.

### Community Strings

Community strings act as plaintext passwords. Three tiers of access:
- `public` — read-only (most common default)
- `private` — read-write
- Custom strings — anything the admin chose

Community strings can be named arbitrarily. When the `public` default is changed, common patterns include:
- Hostname-based: `devicename`, `router01`
- Location-based: `datacenter`, `london`
- Patterns that can be brute-forced with targeted wordlists

### MIB and OID

**MIB** (Management Information Base): a text file defining all queryable objects for a device, organized as a tree hierarchy.

**OID** (Object Identifier): a numeric address for each object in the MIB tree. Longer OIDs are more specific. Example: `.1.3.6.1.2.1.1.1.0` = system description.

Key OID paths:

| OID | Description |
|-----|-------------|
| `.1.3.6.1.2.1.1` | System info (hostname, description, uptime) |
| `.1.3.6.1.2.1.25.1` | Host resources (CPU, memory, processes) |
| `.1.3.6.1.2.1.25.6` | Installed software |
| `.1.3.6.1.2.1.4` | IP routing table |
| `.1.3.6.1.2.1.6` | TCP connections |

### Dangerous SNMP Settings

| Setting | Risk |
|---------|------|
| `rwuser noauth` | Full OID tree access without authentication |
| `rwcommunity <string> <IP>` | Full read/write access from any source |
| `rwcommunity6 <string> <IPv6>` | Same via IPv6 |

## Commands / Syntax

```bash
# Install tools
sudo apt install onesixtyone snmpwalk braa

# Brute-force community strings with onesixtyone
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.14.128

# Walk the full MIB tree (v2c, community string 'public')
snmpwalk -v2c -c public 10.129.14.128

# Walk a specific OID subtree
snmpwalk -v2c -c public 10.129.14.128 .1.3.6.1.2.1.1

# snmpwalk with v1
snmpwalk -v1 -c public 10.129.14.128

# braa — fast OID brute-force (once community string is known)
sudo apt install braa
braa public@10.129.14.128:.1.3.6.*

# braa syntax
braa <community_string>@<IP>:<OID_prefix>
```

## Flags & Options

### snmpwalk

| Flag | Description |
|------|-------------|
| `-v2c` | Use SNMPv2c |
| `-v1` | Use SNMPv1 |
| `-c <string>` | Community string |
| `-O n` | Numeric OID output |
| `-O e` | Don't show errors |

### onesixtyone

| Flag | Description |
|------|-------------|
| `-c <wordlist>` | Community string wordlist |
| `-i <file>` | File of target IPs |
| `-o <file>` | Output file |
| `-t <seconds>` | Timeout |
| `-w <milliseconds>` | Wait between requests |

## Version Detection & Exploit Research

SNMP is unique: the protocol version is chosen by the client during the request, and the community string acts as the access token. Once a community string is known, the MIB tree itself reveals the OS version, kernel, installed packages, and running processes — SNMP is often a better inventory source than the OS itself. The SNMP agent version matters less than the software it runs on (the underlying OS/network device), which is what the MIB data reveals.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap SNMP scan | `nmap -sU -sV -p161 <IP>` | SNMP version support, community string probe |
| System description OID | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.1.1.0` | Full sysDescr — OS, kernel version, hardware |
| Installed software | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.25.6.3.1.2` | All installed package names and versions |
| Running processes | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.25.4.2.1.2` | All running process names |
| System OID tree | `snmpwalk -v2c -c public <IP> .1.3.6.1.2.1.1` | Hostname, OS, uptime, contact, location |
| Nmap SNMP scripts | `nmap -sU -p161 --script snmp-info,snmp-sysdescr <IP>` | System description + info without full walk |

**Key OIDs for version detection:**

| OID | Description |
|-----|-------------|
| `.1.3.6.1.2.1.1.1.0` | `sysDescr` — OS, kernel, hardware model |
| `.1.3.6.1.2.1.1.2.0` | `sysObjectID` — vendor-specific OID tree identifier |
| `.1.3.6.1.2.1.25.6.3.1.2` | `hrSWInstalledName` — installed software list |
| `.1.3.6.1.4.1.9.2.1.3` | Cisco IOS version (Cisco-specific) |

### Searching for Exploits

```bash
# Once sysDescr reveals OS/software:
searchsploit <software> <version>

# SNMP agent vulnerabilities
searchsploit snmp
searchsploit net-snmp

# For network devices:
searchsploit cisco ios <version>
searchsploit juniper <version>

# Metasploit
msf6> search type:exploit name:snmp
```

### Notable CVEs

| CVE | Affected | Impact |
|-----|----------|--------|
| CVE-2017-6736 | Cisco IOS (multiple) | SNMP RCE — crafted SNMP packet triggers buffer overflow; no auth beyond community string |
| CVE-2017-6738 | Cisco IOS XE | SNMP buffer overflow — RCE via SNMP request |
| CVE-2002-0013 | net-snmp < 5.0 | SNMPv1 parsing RCE — unauthenticated |
| CVE-2002-0012 | Multiple vendors | SNMPv1 trap handling heap overflow |
| `public` community (misconfiguration) | Any SNMPv1/v2c | Full MIB read access — OS version, network topology, credentials in config OIDs |

## Gotchas & Notes

- **UDP**: SNMP runs over UDP — nmap needs `-sU` for discovery. Standard `-sV` scans miss it.
- **Community strings are named by administrators** and can follow organizational patterns. If the standard wordlists fail, build custom ones using crunch or cewl based on discovered hostnames and organization names.
- **snmpwalk output is enormous** on a real system — pipe through `grep` for specific targets (IP addresses, usernames, installed packages).
- **Installed packages** are visible in the OID tree — this can reveal vulnerable software versions without touching the application directly.
- **SNMPv3 does not mean secure**: if the implementation has weaknesses in the authentication mechanism, or if SNMPv1/v2c is still enabled alongside v3, the older versions are still exploitable.
- **SNMP write access** (`rwcommunity`) allows setting values — potentially changing device configuration remotely.
- The braa tool is faster than snmpwalk for bulk OID enumeration after a community string is known.

## Related Pages

- [[tools/snmpwalk]]
- [[tools/onesixtyone]]
- [[enumeration/_overview]]

## Sources

- raw/footprinting/host_based_enumeration_snmp.md
