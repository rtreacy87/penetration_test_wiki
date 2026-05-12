---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# snmpwalk

snmpwalk queries an SNMP agent and walks the MIB tree from a given OID, returning all values in the subtree.

## Overview

snmpwalk is part of the `net-snmp` package. Given a valid community string, it iterates through OIDs starting from a base OID, displaying each object and its value. Output from a misconfigured device (using community string `public`) can reveal OS version, installed packages, running processes, network interfaces, and system contact info.

## Commands / Syntax

```bash
# Walk the full MIB tree (SNMPv2c, community string 'public')
snmpwalk -v2c -c public 10.129.14.128

# Walk from specific OID subtree (system info)
snmpwalk -v2c -c public 10.129.14.128 .1.3.6.1.2.1.1

# Walk host resources (installed packages, processes)
snmpwalk -v2c -c public 10.129.14.128 .1.3.6.1.2.1.25

# Walk with custom community string
snmpwalk -v2c -c private 10.129.14.128

# Use SNMPv1
snmpwalk -v1 -c public 10.129.14.128

# Numeric OID output only
snmpwalk -v2c -c public -O n 10.129.14.128

# Filter output for specific content
snmpwalk -v2c -c public 10.129.14.128 | grep -i "package\|install\|software"
snmpwalk -v2c -c public 10.129.14.128 | grep -i "user\|login\|account"
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-v1` | Use SNMPv1 |
| `-v2c` | Use SNMPv2c (most common) |
| `-v3` | Use SNMPv3 (requires auth flags) |
| `-c <string>` | Community string |
| `-O n` | Numeric OID output |
| `-O e` | Don't show SNMP errors |
| `-O q` | Quick/quiet — minimal formatting |
| `-O f` | Full OID output |
| `-t <seconds>` | Timeout (default 5) |
| `-r <retries>` | Number of retries (default 5) |

## Key OID Subtrees

| OID | Content |
|-----|---------|
| `.1.3.6.1.2.1.1` | System description, contact, hostname, uptime |
| `.1.3.6.1.2.1.25.1` | Host resources (uptime, date, processes, storage) |
| `.1.3.6.1.2.1.25.4` | Running processes (process names and IDs) |
| `.1.3.6.1.2.1.25.6` | Installed software/packages |
| `.1.3.6.1.2.1.4` | IP and routing table |
| `.1.3.6.1.2.1.6` | TCP connections and states |
| `.1.3.6.1.2.1.2` | Network interfaces |

## Gotchas & Notes

- Output from a full walk can be thousands of lines — always pipe to grep or tee to a file.
- Installed packages (`.1.3.6.1.2.1.25.6.3.1.2.*`) reveal the complete software inventory including version numbers — useful for identifying vulnerable software without touching the application.
- Running processes (`.1.3.6.1.2.1.25.4`) show what services are active, including internal services not exposed on the network.
- If no community string is known, use [[tools/enumeration/onesixtyone]] first to brute-force it, then use snmpwalk.
- System contact (OID `.1.3.6.1.2.1.1.4.0`) often contains an email address — useful for staff enumeration.

## Installation

```bash
# Check if installed
snmpwalk --version 2>/dev/null | head -1 || echo "not installed"

# Install — part of the snmp package
sudo apt install snmp snmp-mibs-downloader -y

# Verify
snmpwalk --version
```

> **Note:** `snmp-mibs-downloader` provides MIB files for human-readable OID names. Without it, snmpwalk output uses numeric OIDs only.

## Related Pages

- [[enumeration/snmp]]
- [[tools/enumeration/onesixtyone]]

## Sources

- raw/footprinting/host_based_enumeration_snmp.md
