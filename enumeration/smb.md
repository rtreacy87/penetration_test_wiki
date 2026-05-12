---
tags: [enumeration, enumeration/smb]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# SMB Enumeration

SMB enumeration: shares, null sessions, user/group enumeration via rpcclient, smbclient, enum4linux, and crackmapexec.

## Overview

SMB (Server Message Block) is Microsoft's file/printer/resource sharing protocol. It runs on TCP ports **139** (NetBIOS) and **445** (direct TCP). Linux implementations use **Samba** (which also implements CIFS — SMB version 1 dialect).

SMB is a primary target during internal network assessments because:
- Null sessions (unauthenticated access) are often still possible on Samba servers
- Shares frequently contain sensitive files (configs, credentials, backups)
- User and group enumeration via RPC provides targets for further attacks

## Key Concepts / Techniques

### SMB Versions

| Version | OS Support | Notable Features |
|---------|-----------|-----------------|
| CIFS (SMB 1) | Windows NT 4.0 | NetBIOS-based; outdated and insecure |
| SMB 2.0 | Windows Vista/Server 2008 | Performance improvements, message signing |
| SMB 2.1 | Windows 7/Server 2008 R2 | Locking improvements |
| SMB 3.0 | Windows 8/Server 2012 | Multichannel, end-to-end encryption |
| SMB 3.1.1 | Windows 10/Server 2016 | AES-128 encryption, integrity checking |

### Null Sessions

A null session is an anonymous (unauthenticated) connection to the IPC$ share. When a Samba server is configured with `map to guest = bad user` and `usershare allow guests = yes`, null sessions work. This allows:
- Share enumeration
- User enumeration (via RPC)
- Group/policy enumeration

### Key Samba Dangerous Settings

| Setting | Description |
|---------|-------------|
| `browseable = yes` | Share appears in network listing |
| `read only = no` | Files can be written |
| `writable = yes` | Files can be created/modified |
| `guest ok = yes` | No password needed |
| `enable privileges = yes` | Respects SID-based privileges |
| `create mask = 0777` | New files world-readable/writable |
| `map to guest = bad user` | Invalid users connect as guest |

### RPC Enumeration

The Remote Procedure Call (RPC) interface (accessed via `rpcclient`) exposes user, group, share, and domain information. It is the primary unauthenticated enumeration interface on SMB servers.

RID brute-forcing: enumerate users by querying every RID from 500 to ~1100 to find user accounts not returned by `enumdomusers`.

## Commands / Syntax

```bash
# Nmap SMB scan
sudo nmap 10.129.14.128 -sV -sC -p139,445

# List shares with null session (no credentials)
smbclient -N -L //10.129.14.128

# Connect to a share
smbclient //10.129.14.128/notes
# Then use: ls, get <file>, !ls, !cat <file>

# Download file from share
smb: \> get prep-prod.txt

# smbmap — show share permissions
smbmap -H 10.129.14.128

# CrackMapExec — enumerate shares with null session
crackmapexec smb 10.129.14.128 --shares -u '' -p ''

# rpcclient — null session RPC
rpcclient -U "" 10.129.14.128
Enter WORKGROUP\'s password: (press Enter)

# rpcclient commands
rpcclient$> srvinfo                         # Server info
rpcclient$> enumdomains                     # List domains
rpcclient$> querydominfo                    # Domain/server/user info
rpcclient$> netshareenumall                 # All shares
rpcclient$> netsharegetinfo <share>         # Specific share info
rpcclient$> enumdomusers                    # Domain users
rpcclient$> queryuser <RID>                 # User details by RID (e.g., 0x3e9)
rpcclient$> querygroup 0x201               # Group info by RID

# RID brute-force to enumerate users
for i in $(seq 500 1100); do
    rpcclient -N -U "" 10.129.14.128 -c "queryuser 0x$(printf '%x\n' $i)" \
    | grep "User Name\|user_rid\|group_rid" && echo ""
done

# Impacket samrdump — enumerate via SAMR protocol
samrdump.py 10.129.14.128

# enum4linux-ng — comprehensive automated enumeration
./enum4linux-ng.py 10.129.14.128 -A
```

## Flags & Options

### smbclient

| Flag | Description |
|------|-------------|
| `-N` | No password (null session) |
| `-L //host` | List shares |
| `-U <user>` | Specify username |
| `--password <pw>` | Specify password |

### crackmapexec smb

| Flag | Description |
|------|-------------|
| `--shares` | Enumerate shares |
| `-u ''` | Blank username |
| `-p ''` | Blank password |
| `--users` | Enumerate users |
| `--groups` | Enumerate groups |
| `--rid-brute` | RID brute-force |

### enum4linux-ng

| Flag | Description |
|------|-------------|
| `-A` | All checks |
| `-U` | Users |
| `-G` | Groups |
| `-S` | Shares |
| `-P` | Password policy |
| `-N` | NetBIOS names |

## Version Detection & Exploit Research

SMB version and OS information are exposed unauthenticated via the SMB negotiation handshake. Nmap's `smb-os-discovery` and `smb2-security-mode` scripts, as well as CrackMapExec and Metasploit's `smb_version` module, all extract this without credentials. SMB has produced some of the most critical CVEs in Windows history — EternalBlue and BlueKeep both required only network access and produced SYSTEM-level RCE. Version detection is mandatory before moving to exploitation.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap smb-os-discovery | `nmap -p445 --script smb-os-discovery <IP>` | OS, domain, SMB dialect, hostname |
| Nmap smb2-security-mode | `nmap -p445 --script smb2-security-mode <IP>` | SMB2/3 negotiation + signing required flag |
| CrackMapExec | `crackmapexec smb <IP>` | OS version, SMB signing, domain, hostname |
| Metasploit smb_version | `use auxiliary/scanner/smb/smb_version` | OS + SMB version details |
| Nmap all smb scripts | `nmap -p139,445 --script smb-vuln* <IP>` | Direct CVE checks (EternalBlue, MS08-067, etc.) |
| rpcclient srvinfo | `rpcclient -U "" <IP> -c srvinfo` | Server type flags, OS version string |

**What CrackMapExec output reveals:**
`SMB  10.10.10.100  445  WS01  [*] Windows 10.0 Build 19041 x64 (name:WS01) (domain:CORP) (signing:False) (SMBv1:False)`
- `Windows 10.0 Build 19041` → map to Windows version and patch level
- `signing:False` → relay attacks possible (Responder + ntlmrelayx)
- `SMBv1:True` → EternalBlue likely applicable

### Searching for Exploits

```bash
# Searchsploit
searchsploit smb
searchsploit samba
searchsploit "ms17-010"    # EternalBlue
searchsploit "ms08-067"    # NetAPI

# Metasploit — direct CVE checks
msf6> use auxiliary/scanner/smb/smb_ms17_010   # EternalBlue check
msf6> use exploit/windows/smb/ms17_010_eternalblue
msf6> use exploit/windows/smb/ms08_067_netapi
msf6> search type:exploit name:smb
```

### Notable CVEs

| CVE / MS Bulletin | Affected OS | Impact |
|-------------------|------------|--------|
| CVE-2017-0144 (MS17-010, EternalBlue) | Windows 7, Server 2008/2012 (unpatched) | Unauthenticated RCE via SMBv1 — SYSTEM level |
| CVE-2008-4250 (MS08-067) | Windows XP, 2000, 2003, Vista, 2008 | Unauthenticated RCE via NetAPI — SYSTEM level |
| CVE-2020-0796 (SMBGhost) | Windows 10 1903/1909, Server 1903/1909 | SMBv3 compression overflow — unauthenticated RCE |
| CVE-2021-34527 (PrintNightmare) | All Windows with Print Spooler | Auth'd RCE/LPE via spooler service |
| CVE-2017-7494 (SambaCry) | Samba 3.5.0–4.6.4 | Unauthenticated RCE via writable share |
| CVE-2021-44142 | Samba < 4.13.17 | Heap buffer overflow in vfs_fruit — RCE as root |

## Gotchas & Notes

- **Use multiple tools**: rpcclient, smbclient, smbmap, enum4linux-ng, and crackmapexec each return different information. Do not rely on a single tool.
- **Samba's `smb signing required = false`** enables relay attacks. Note this in findings.
- `enumdomusers` via rpcclient may not return all users — always supplement with RID brute-forcing from 500 to 1100.
- **smbstatus** (run on the server) shows active connections including your own — useful during internal assessments to verify you are authenticated as the correct user.
- CrackMapExec output shows share permissions: `READ,WRITE` vs `NO ACCESS` vs blank. Target writable shares.
- `queryuser 0x3e8` is RID 1000 — the first non-default local user on Linux Samba. RID 500 is the built-in Administrator on Windows.
- `IPC$` share is always present and is the mechanism for null session enumeration — you do not need to "connect" to it explicitly; rpcclient handles this.

## Related Pages

- [[attack/smb]] — exploitation: null sessions, password spray, RCE, PTH, Responder, NTLM relay
- [[tools/smbclient]]
- [[tools/rpcclient]]
- [[tools/enum4linux]]
- [[tools/crackmapexec]]
- [[tools/impacket]]
- [[tools/responder]]
- [[enumeration/_overview]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
