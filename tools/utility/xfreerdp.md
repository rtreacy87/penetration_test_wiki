---
tags: [tool]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 2
---

# xfreerdp

Modern, actively maintained RDP client for Linux; the go-to tool for pentesting because it supports NLA, Pass-the-Hash, and current RDP protocol versions.

## Overview

xfreerdp is the FreeRDP project's X11 client. It implements RDP versions 5 through 10 and is the only Linux RDP client that supports:

- **NLA (Network Level Authentication)** — required by default on Windows 7+, Windows Server 2008+
- **Pass-the-Hash** (`/pth`) — authenticate with an NT hash directly, no plaintext password needed
- **Drive sharing** (`/drive`) — mount a local folder inside the remote session for file transfer
- **Clipboard** (`/clipboard`) — copy/paste between attacker and target

In virtually every modern engagement, xfreerdp is the correct choice over rdesktop. See [[tools/utility/rdesktop]] for the legacy alternative and a side-by-side comparison.

## Installation

```bash
# Check if installed
xfreerdp --version 2>/dev/null | head -1 || echo "not installed"

# Install (Kali / Parrot)
sudo apt install freerdp2-x11 -y

# Verify
xfreerdp --version
```

Note: the package is `freerdp2-x11` but the binary is `xfreerdp`. On newer Kali builds you may see `freerdp3-x11` — install whichever is available; the flags are identical.

## Usage

### Basic connection

```bash
xfreerdp /v:10.129.14.128 /u:administrator /p:'password123'
```

### With domain account

```bash
xfreerdp /v:10.129.14.128 /u:htb-rdp /p:'HTBRocks!' /d:INLANEFREIGHT
```

### Suppress certificate prompt (self-signed certs in labs)

```bash
xfreerdp /v:10.129.14.128 /u:administrator /p:'password123' /cert-ignore
```

### Pass-the-Hash (no plaintext password required)

```bash
# Requires DisableRestrictedAdmin = 0 on target first
xfreerdp /v:10.129.14.128 /u:administrator /pth:0E14B9D6330BF16C30B1924111104824 /cert-ignore
```

### Non-standard port

```bash
xfreerdp /v:10.129.14.128:3390 /u:administrator /p:'password123'
```

### Mount a local folder for file transfer

```bash
xfreerdp /v:10.129.14.128 /u:administrator /p:'password123' \
    /drive:share,/tmp/transfer /cert-ignore
# Inside the session: \\tsclient\share maps to /tmp/transfer on your machine
```

### Headless (no physical display)

```bash
Xvfb :99 -screen 0 1024x768x24 &
DISPLAY=:99 xfreerdp /v:10.129.14.128 /u:administrator /p:'password123' /cert-ignore
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `/v:<host>` | Target host IP or hostname. Append `:PORT` for non-standard port (e.g., `/v:10.0.0.1:3390`). |
| `/u:<user>` | Username |
| `/p:<pass>` | Password. Single-quote if it contains special characters. |
| `/pth:<NT_hash>` | Pass-the-Hash: use the NT hash directly instead of a plaintext password. Requires Restricted Admin Mode on the target. |
| `/d:<domain>` | Domain name for Windows/Active Directory authentication |
| `/cert-ignore` | Accept any certificate without prompting — use for lab self-signed certs |
| `/cert-tofu` | Trust On First Use — saves the cert fingerprint and warns if it changes |
| `/drive:<name>,<path>` | Mount a local directory as a shared drive inside the session |
| `/clipboard` | Enable clipboard sync between attacker and target |
| `/dynamic-resolution` | Resize the RDP window and the remote session adjusts automatically |
| `/w:<width> /h:<height>` | Set fixed resolution (e.g., `/w:1920 /h:1080`) |
| `/f` | Fullscreen |
| `/kbd:0x0409` | Set keyboard layout (0x0409 = US English) — use if key mappings are wrong |
| `/sound` | Enable audio redirection |

## Gotchas & Notes

- If the target requires NLA and you get an instant disconnect with no error, the issue is often NLA — xfreerdp handles it automatically, but the account may need to be in the right group
- `/pth` is silently rejected if `DisableRestrictedAdmin` is absent or set to `1` on the target — set it first via an existing shell: `reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f`
- Pass only the NT hash (second half of `LM:NT` output from secretsdump/mimikatz) — passing the full `LM:NT` string or only the LM hash will fail
- xfreerdp exits immediately on connection failure — if you see no output and no window, increase verbosity with `/log-level:DEBUG`
- Self-signed certificates are common in labs; `/cert-ignore` skips the interactive prompt. In real engagements, inspect the cert instead

## Related Pages

- [[tools/utility/rdesktop]] — legacy alternative; simpler syntax but no NLA or PTH
- [[attack/rdp]] — full RDP attack reference: spraying, session hijacking, PTH, BlueKeep
- [[labs/htb/attacking_common_services/rdp_pass_the_hash]] — lab walkthrough using xfreerdp PTH
- [[enumeration/windows_remote_mgmt]] — RDP enumeration and fingerprinting

## Sources

- raw/attacking_common_services/rdp_attacking_rdp.md
- raw/attacking_common_services/rdp_latest_rdp_vulnerabilities.md
