---
tags: [lab]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# HTB Footprinting Lab — Medium

HTB Academy Footprinting module medium lab: internal server accessible to all network users — enumerate services, obtain credentials for the HTB user.

## Target Info

- **Client**: Inlanefreight Ltd
- **Target**: Internal server accessible to the entire internal network
- **Constraint**: No aggressive exploitation
- **Goal**: Obtain credentials for user `HTB`
- **Proof**: HTB user credentials

## Approach / Methodology

"A server that everyone on the internal network has access to" strongly suggests a file server or infrastructure server with broadly accessible services — likely NFS, SMB, FTP, or a combination. The HTB user is created specifically for this assessment.

### Phase 1: Port Scan

```bash
# Full port scan
sudo nmap -sV -sC -p- --open 10.129.x.x

# UDP scan for SNMP
sudo nmap -sU -p161 10.129.x.x
```

Expected services on an "accessible to all" internal server:
- TCP 21 (FTP)
- TCP 22 (SSH)
- TCP 111 / 2049 (NFS/RPC)
- TCP 139 / 445 (SMB)
- UDP 161 (SNMP)
- TCP 3306 (MySQL, if database-backed)

### Phase 2: FTP Enumeration

An "accessible to all" server frequently has anonymous FTP:

```bash
# Check for anonymous access
ftp 10.129.x.x
# Username: anonymous, Password: (blank)

# If anonymous works — recursive listing
ftp> ls -R

# Download all files
wget -m --no-passive ftp://anonymous:anonymous@10.129.x.x
```

Look for credentials, SSH keys, or configuration files in FTP directories.

### Phase 3: SMB Enumeration

```bash
# List shares (null session)
smbclient -N -L //10.129.x.x

# Enumerate everything
./enum4linux-ng.py 10.129.x.x -A

# Check share permissions
nxc smb 10.129.x.x --shares -u '' -p ''

# Access readable shares
smbclient -N //10.129.x.x/sharename
```

### Phase 4: NFS Enumeration

```bash
# Find NFS exports
showmount -e 10.129.x.x

# Mount and explore
mkdir target-NFS
sudo mount -t nfs 10.129.x.x:/ ./target-NFS/ -o nolock
ls -ln target-NFS/    # Check UIDs for access control
```

### Phase 5: SNMP Enumeration

```bash
# Brute-force community string
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt 10.129.x.x

# Walk the MIB tree with found community string
snmpwalk -v2c -c <community_string> 10.129.x.x

# Look for credentials or useful info
snmpwalk -v2c -c public 10.129.x.x | grep -i "user\|pass\|login\|htb"
```

SNMP can reveal running processes, installed software, and sometimes credentials stored in process arguments.

### Phase 6: Correlate Findings

- Use any credentials found (FTP config files, SMB share content, SNMP process args) to authenticate to other services.
- The HTB user credentials may be in a file on an FTP/SMB share, in the SNMP output, or may require combining multiple findings.
- Test found credentials across SSH, FTP, SMB, and any web interface.

## Key Techniques Used

1. **Multi-service enumeration** — FTP, SMB, NFS, SNMP all queried in parallel
2. **Anonymous/null session access** — reduces credential requirements
3. **SNMP MIB walking** — reveals running processes and may expose credentials
4. **Credential correlation** — credentials found on one service tested against all others

## Lessons Learned

- "Accessible to all" implies intentional guest/anonymous access — check FTP anonymous, SMB null sessions, and NFS without auth first.
- SNMP is frequently overlooked (UDP, not TCP) — always include UDP 161 in internal scans.
- A server that "everyone accesses" likely has broad read permissions on some shares — mount everything and search for the HTB user's credentials.
- Administrators often store credentials in files on file shares that "only internal users can access" — this is not security.

## Related Pages

- [[enumeration/ftp]]
- [[enumeration/smb]]
- [[enumeration/nfs]]
- [[enumeration/snmp]]
- [[tools/enumeration/smbclient]]
- [[tools/enumeration/onesixtyone]]
- [[tools/enumeration/snmpwalk]]

## Sources

- raw/footprinting/footprinti0ng_lab_medium.md
