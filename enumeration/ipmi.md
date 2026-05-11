---
tags: [enumeration, enumeration/ipmi]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# IPMI Enumeration

IPMI enumeration: BMC discovery, default credentials, RAKP hash retrieval via Cipher 0 flaw, and Hashcat cracking.

## Overview

IPMI (Intelligent Platform Management Interface) is a hardware-level management system built into server motherboards. It operates independently of the host OS — you can access it when the server is powered off, unresponsive, or in BIOS setup. It communicates over **UDP port 623**.

The physical component is the **BMC (Baseboard Management Controller)** — typically an embedded ARM chip running Linux. BMCs expose web consoles, SSH/Telnet, and the IPMI protocol directly.

Getting into a BMC is equivalent to physical access: you can power cycle the machine, reinstall the OS, read serial console output, and view/change BIOS settings.

## Key Concepts / Techniques

### Common BMC Implementations

| Product | Default Username | Default Password |
|---------|-----------------|-----------------|
| Dell iDRAC | root | calvin |
| HP iLO | Administrator | Randomized 8-char (uppercase + numbers) |
| Supermicro IPMI | ADMIN | ADMIN |

Always try default credentials first. Many BMCs ship with these and are never changed.

### RAKP Protocol Flaw (IPMI 2.0)

The most significant IPMI vulnerability: During authentication in IPMI 2.0, **the server sends a salted SHA1 or MD5 hash of the user's password to the client before authentication is complete**. This hash can be captured for any valid user account.

This is not a typical MITM — you send a legitimate authentication request and the server responds with the hash. The hash can then be cracked offline using Hashcat mode `7300`.

This works for any valid user account on the BMC, regardless of password complexity. The only limitation is cracking the hash — for HP iLO with its default randomized 8-character password, use a mask attack:

```
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

This tries all combinations of uppercase letters and digits for 8-character passwords.

### Why IPMI Matters

- Default credentials on BMCs are extremely common
- IPMI password reuse: organizations sometimes use a single BMC password across many servers — one cracked hash unlocks everything
- BMC access = full server control even without OS credentials

## Commands / Syntax

```bash
# Nmap scan — UDP port 623
sudo nmap -sU --script ipmi-version -p623 ilo.inlanfreight.local

# Metasploit — discover IPMI version
msf6 > use auxiliary/scanner/ipmi/ipmi_version
msf6 auxiliary(scanner/ipmi/ipmi_version) > set rhosts 10.129.42.195
msf6 auxiliary(scanner/ipmi/ipmi_version) > run

# Metasploit — dump IPMI hashes (RAKP flaw)
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > set rhosts 10.129.42.195
msf6 auxiliary(scanner/ipmi/ipmi_dumphashes) > run

# Crack the hash with Hashcat (mode 7300)
hashcat -m 7300 ipmi_hash.txt /usr/share/wordlists/rockyou.txt

# Crack HP iLO default format (8-char uppercase+digits)
hashcat -m 7300 ipmi_hash.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u
```

## Flags & Options

### Metasploit ipmi_dumphashes Module Options

| Option | Description |
|--------|-------------|
| `RHOSTS` | Target IP(s) |
| `RPORT` | Default 623 |
| `CRACK_COMMON` | Auto-crack common passwords as hashes are obtained |
| `OUTPUT_HASHCAT_FILE` | Save hashes in hashcat format |
| `OUTPUT_JOHN_FILE` | Save hashes in john format |
| `PASS_FILE` | Wordlist for auto-cracking |
| `USER_FILE` | Usernames to probe |
| `THREADS` | Max one per host |

### Hashcat IPMI Mode

| Mode | Hash Type |
|------|-----------|
| 7300 | IPMI2 RAKP HMAC-SHA1 |

## Gotchas & Notes

- **UDP only**: IPMI runs on UDP 623. Nmap's standard TCP scan won't find it. Always include `-sU -p623` in internal network scans.
- **No fix for the RAKP flaw**: This is a design flaw in the IPMI specification, not a bug. There is no patch. The only mitigations are strong passwords and network segmentation.
- **Password reuse is the jackpot**: When a cracked IPMI password matches the root password or domain account on the same server, you get a massive privilege escalation.
- **HP iLO randomized passwords** are hard to crack by default, but if you find one, cracked or not — check for other vectors (web console access, known CVEs in the iLO firmware version).
- **Common password list in Metasploit**: The `ipmi_passwords.txt` wordlist is worth knowing — it contains vendor default passwords and common variants.
- **BMC access findings are critical severity**: document as high/critical regardless of what you find inside — the access itself is a significant finding.

## Related Pages

- [[enumeration/_overview]]

## Sources

- raw/footprinting/host_based_enumeration_ipmi.md
