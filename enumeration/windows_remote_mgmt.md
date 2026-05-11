---
tags: [enumeration, protocol]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Windows Remote Management Protocols

WinRM, RDP, and WMI: enumeration, footprinting, and authentication testing on Windows remote management services.

## Overview

Windows servers expose three primary remote management protocols:
- **RDP (Remote Desktop Protocol)**: TCP 3389 — GUI access
- **WinRM (Windows Remote Management)**: TCP 5985 (HTTP) / 5986 (HTTPS) — PowerShell remoting, command execution
- **WMI (Windows Management Instrumentation)**: TCP 135 + dynamic port — script/API-based management

These services are heavily targeted during internal assessments because they provide direct interactive access to Windows systems. Credential discovery anywhere in the network should immediately be tested against these services.

## Key Concepts / Techniques

### RDP — TCP 3389

Microsoft's proprietary GUI remote access protocol. Supports NLA (Network Level Authentication) which requires credentials before the full RDP session is established.

**Security notes:**
- Default certificates are self-signed — clients cannot verify server identity
- Without NLA, the login screen is exposed pre-auth (usable for username enumeration)
- `--packet-trace` in nmap reveals the `mstshash=nmap` cookie — detectable by EDR/threat hunters

**RDP Security Layers:**
- `PROTOCOL_RDP` — Basic RDP encryption (weak)
- `PROTOCOL_SSL` — TLS
- `PROTOCOL_HYBRID` — NLA (CredSSP) — strongest

### WinRM — TCP 5985/5986

SOAP-based protocol using WS-Management standard. Enabled by default on Windows Server 2012+. WinRS (Windows Remote Shell) executes arbitrary commands.

**evil-winrm** is the go-to Linux client for WinRM. Provides a full PowerShell session when valid credentials are provided.

**Common WinRM findings:**
- Port 5985 open but not requiring HTTPS (credentials in plaintext)
- Weak credentials that pass `Test-WsMan` checks
- Default service accounts with remote management rights

### WMI — TCP 135 + dynamic

WMI provides read/write access to virtually all Windows system settings. Initialized on port 135; subsequent communication moves to a random port. Impacket's `wmiexec.py` provides command execution.

**WMI uses:** PowerShell (Get-WmiObject), VBScript, WMIC command, and tools like wmiexec from Impacket.

## Commands / Syntax

```bash
# RDP — Nmap scan with RDP scripts
nmap -sV -sC 10.129.201.248 -p3389 --script rdp*

# RDP — Check security settings with rdp-sec-check
git clone https://github.com/CiscoCXSecurity/rdp-sec-check.git && cd rdp-sec-check
./rdp-sec-check.pl 10.129.201.248

# RDP — Connect from Linux with xfreerdp
xfreerdp /u:cry0l1t3 /p:"P455w0rd!" /v:10.129.201.248

# RDP — Connect with rdesktop
rdesktop -u cry0l1t3 -p "P455w0rd!" 10.129.201.248

# WinRM — Nmap scan
nmap -sV -sC 10.129.201.248 -p5985,5986 --disable-arp-ping -n

# WinRM — Test connectivity (PowerShell, Windows)
Test-WsMan 10.129.201.248

# WinRM — Connect with evil-winrm (Linux)
evil-winrm -i 10.129.201.248 -u Cry0l1t3 -p P455w0rD!

# evil-winrm — with certificate
evil-winrm -i 10.129.201.248 -u username -c certificate.pem -k key.pem -S

# WMI — command execution with wmiexec.py (Impacket)
/usr/share/doc/python3-impacket/examples/wmiexec.py Cry0l1t3:"P455w0rD!"@10.129.201.248 "hostname"

# WMI — pass-the-hash
wmiexec.py -hashes :NTHASH Administrator@10.129.201.248 "cmd.exe /c whoami"
```

## Flags & Options

### Nmap RDP Scripts

| Script | Description |
|--------|-------------|
| `rdp-enum-encryption` | Tests which encryption layers are supported |
| `rdp-ntlm-info` | Retrieves domain, hostname, product version via NTLM |
| `rdp-vuln-ms12-020` | Tests for MS12-020 DoS vulnerability |

### xfreerdp

| Flag | Description |
|------|-------------|
| `/u:<user>` | Username |
| `/p:<pass>` | Password |
| `/v:<host>` | Target host |
| `/d:<domain>` | Domain |
| `/dynamic-resolution` | Resize window |
| `+clipboard` | Enable clipboard sharing |
| `/cert-ignore` | Ignore certificate errors |

### evil-winrm

| Flag | Description |
|------|-------------|
| `-i <IP>` | Target IP |
| `-u <user>` | Username |
| `-p <pass>` | Password |
| `-H <hash>` | NTLM hash (pass-the-hash) |
| `-S` | Use HTTPS (port 5986) |
| `-c <cert>` | SSL certificate |
| `-k <key>` | SSL private key |

## Gotchas & Notes

- **NLA fingerprinting**: `CredSSP (NLA): SUCCESS` in nmap output means NLA is required — credentials must be valid before the full desktop session starts. This prevents the traditional credential brute-force via RDP login screen.
- **RDP cookies are detectable**: Nmap's RDP scripts send `mstshash=nmap`. Threat hunters and EDR solutions flag this. Be aware on hardened/monitored networks.
- **WinRM on HTTP (5985)**: Credentials are sent base64-encoded but not encrypted. This is a finding. Prefer 5986 (HTTPS).
- **evil-winrm gives a full PS session**: Once connected, you can upload/download files, bypass AMSI, and run PowerShell scripts directly.
- **WMI dynamic ports**: The firewall must permit the initial 135/tcp connection plus subsequent dynamic ports. WMI may be blocked by host firewalls even if 135 is open.
- **wmiexec.py uses SMBv3**: The wmiexec tool actually uses SMB for file output — note the `SMBv3.0 dialect used` message in output. This means SMB port 445 must also be accessible.
- **Pass-the-hash**: All three protocols support pass-the-hash attacks with the right tooling. RDP requires `DisableRestrictedAdmin = 0` on the target.
- **RDP brute-force detection**: Most Windows systems with NLA will lock accounts after failed attempts. Check the account lockout policy before brute-forcing RDP.

## Related Pages

- [[enumeration/_overview]]
- [[enumeration/linux_remote_mgmt]]
- [[enumeration/smb]]
- [[tools/crackmapexec]]
- [[tools/impacket]]

## Sources

- raw/footprinting/windows_remote_management_protocols.md
