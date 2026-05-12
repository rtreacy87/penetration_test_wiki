---
tags: [tool]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 2
---

# rdesktop

Legacy Linux RDP client with simple syntax; only relevant for very old Windows targets (XP, Server 2003) where NLA is not enforced.

## Overview

rdesktop is one of the original open-source RDP clients for Linux. It implements RDP up to version 5 — the protocol version used by Windows XP and Windows Server 2003. It has not been actively maintained since roughly 2008.

**In almost all modern engagements, use [[tools/utility/xfreerdp]] instead.** rdesktop will fail or refuse to connect to any Windows system that requires NLA (the default since Windows Vista/Server 2008), and it cannot perform Pass-the-Hash. It remains documented here because legacy infrastructure still exists and its simpler syntax occasionally appears in older write-ups and course material.

See the comparison table at the bottom of this page for a full side-by-side.

## Installation

```bash
# Check if installed
rdesktop --version 2>/dev/null | head -1 || echo "not installed"

# Install (Kali / Parrot)
sudo apt install rdesktop -y

# Verify
rdesktop --version
```

## Usage

### Basic connection

```bash
rdesktop -u admin -p password123 10.129.14.128
```

On first connect to an unknown host, rdesktop prompts you to accept the server certificate. Type `yes` and press Enter.

### With domain

```bash
rdesktop -u admin -p password123 -d INLANEFREIGHT 10.129.14.128
```

### Custom resolution

```bash
rdesktop -u admin -p password123 -g 1280x800 10.129.14.128
```

### Fullscreen

```bash
rdesktop -u admin -p password123 -f 10.129.14.128
```

### Non-standard port

```bash
rdesktop -u admin -p password123 10.129.14.128:3390
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-u <user>` | Username |
| `-p <pass>` | Password |
| `-d <domain>` | Windows domain for authentication |
| `-g <WxH>` | Screen resolution (e.g., `-g 1280x1024`). Default is 800x600. |
| `-f` | Fullscreen |
| `-k <keymap>` | Keyboard layout (e.g., `-k en-us`). Use if keys map incorrectly. |
| `-r disk:<name>=<path>` | Share a local directory inside the session (e.g., `-r disk:share=/tmp`) |
| `-r clipboard:PRIMARYCLIPBOARD` | Enable clipboard sync |
| `-T <title>` | Set window title |
| `-P` | Enable persistent bitmap caching — speeds up redraws on slow links |

## rdesktop vs xfreerdp — When to Use Which

| Capability | rdesktop | xfreerdp |
|------------|----------|----------|
| **NLA support** | No — will fail on Windows Vista+ with NLA enabled | Yes — handles NLA automatically |
| **RDP protocol version** | Up to RDP 5 (Windows XP era) | RDP 5 through 10 (current) |
| **Pass-the-Hash** | No | Yes (`/pth`) |
| **Windows XP / Server 2003** | Works | Also works |
| **Windows 7 / Server 2008+** | Fails if NLA enforced (usually) | Works |
| **Active maintenance** | No (effectively abandoned) | Yes (FreeRDP project) |
| **Syntax style** | Simple: `-u user -p pass host` | Verbose: `/v:host /u:user /p:pass` |
| **Drive/clipboard sharing** | Basic (`-r disk:`, `-r clipboard:`) | Full support (`/drive:`, `/clipboard`) |
| **Headless operation** | Limited | Supported via Xvfb |

**Decision rule:** Use xfreerdp unless you have confirmed the target is Windows XP or Server 2003 with NLA disabled and you specifically prefer rdesktop's simpler flag syntax. For any modern Windows target or any PTH scenario, xfreerdp is the only option.

## Gotchas & Notes

- rdesktop cannot negotiate NLA — if the target requires it (the default on any modern Windows), the connection fails immediately, often with a cryptic TLS error
- No `/pth` equivalent exists — hash-based authentication is not supported
- The certificate acceptance prompt on first connect is interactive; it cannot be suppressed with a flag (unlike xfreerdp's `/cert-ignore`)
- rdesktop-based write-ups in older courses often predate Windows Vista becoming the baseline — treat them as historical context and translate the flags to xfreerdp equivalents

## Related Pages

- [[tools/utility/xfreerdp]] — modern replacement; use this for all current engagements
- [[attack/rdp]] — full RDP attack reference: spraying, session hijacking, PTH, BlueKeep
- [[enumeration/windows_remote_mgmt]] — RDP enumeration and fingerprinting

## Sources

- raw/attacking_common_services/rdp_attacking_rdp.md
- raw/attacking_common_services/rdp_latest_rdp_vulnerabilities.md
