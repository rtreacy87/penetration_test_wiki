---
tags: [wordlist, reference]
module: multi
last_updated: 2026-05-11
source_count: 0
---

# Wordlist Use Cases

Per-tool and per-service wordlist selection. All paths relative to `/usr/share/seclists/` unless otherwise noted.

## Overview

Choosing the right wordlist is a trade-off between coverage (larger lists) and speed (smaller lists). The pattern is always: start small → escalate to medium → targeted custom list if still missing. Default credential lists should always be tried before brute-force.

Base path shorthand used below:
- `$SL` = `/usr/share/seclists` — [github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists)
- `$WL` = `/usr/share/wordlists`

> **rockyou.txt ships gzipped on Kali and Parrot.** Run `gunzip /usr/share/wordlists/rockyou.txt.gz` before first use, otherwise all `$WL/rockyou.txt` references below will fail. Alternatively, extract from SecLists: `tar -xzf $SL/Passwords/Leaked-Databases/rockyou.txt.tar.gz -C $WL/`

---

## By Tool

### Hydra

Hydra needs a username source and a password source. Use `-L` for a list file, `-l` for a single value. The wordlist choice changes by service.

| Service | Username List | Password List |
|---------|--------------|---------------|
| FTP | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| SSH | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| Telnet | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| HTTP Basic/Digest | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| HTTP POST form | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| SMTP AUTH | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| SMB | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| MySQL | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| MSSQL | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| RDP | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| VNC | *(no username)* | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| IMAP/POP3 | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |

```bash
# FTP brute-force
hydra -L $SL/Usernames/top-usernames-shortlist.txt \
      -P $WL/rockyou.txt \
      ftp://10.10.10.10

# SSH brute-force (slow due to protocol overhead — use targeted lists)
hydra -l root \
      -P $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
      ssh://10.10.10.10

# HTTP POST form
hydra -L $SL/Usernames/top-usernames-shortlist.txt \
      -P $WL/rockyou.txt \
      10.10.10.10 http-post-form \
      "/login:username=^USER^&password=^PASS^:Invalid credentials"

# SMB — password spray (careful of lockout)
hydra -L usernames.txt \
      -P $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
      smb://10.10.10.10

# SMTP authentication
hydra -L $SL/Usernames/top-usernames-shortlist.txt \
      -P $WL/rockyou.txt \
      smtp://10.10.10.10

# MySQL
hydra -l root \
      -P $WL/rockyou.txt \
      mysql://10.10.10.10

# RDP (use small password list — lockout risk is high)
hydra -L $SL/Usernames/top-usernames-shortlist.txt \
      -P $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
      rdp://10.10.10.10
```

---

### Medusa

Similar to Hydra. Supports parallel attacks across multiple hosts.

| Service | Username List | Password List |
|---------|--------------|---------------|
| FTP | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| SSH | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| SMB | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| HTTP | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |

```bash
# FTP brute-force
medusa -h 10.10.10.10 \
       -U $SL/Usernames/top-usernames-shortlist.txt \
       -P $WL/rockyou.txt \
       -M ftp

# SMB — parallel across /24 subnet
medusa -H targets.txt \
       -U $SL/Usernames/top-usernames-shortlist.txt \
       -P $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
       -M smbnt -T 10
```

---

### NetExec

CME is used for credential validation and password spraying across SMB, WinRM, MSSQL, SSH, and LDAP. It is not a traditional brute-forcer — use it for spraying known passwords across many hosts or validating credentials found elsewhere.

| Use Case | Username Source | Password Source |
|----------|----------------|----------------|
| Password spray (SMB) | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| Credential validation | Single user | Single password or small list |
| User enumeration (RID brute) | *(no list needed)* | *(no list needed)* |

```bash
# SMB password spray
nxc smb 10.10.10.0/24 \
             -u $SL/Usernames/top-usernames-shortlist.txt \
             -p $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
             --no-bruteforce  # try each user with each password, not all combinations

# WinRM spray
nxc winrm 10.10.10.10 \
             -u usernames.txt \
             -p passwords.txt

# MSSQL
nxc mssql 10.10.10.10 \
             -u sa \
             -p $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt
```

---

### Gobuster

Gobuster handles directory, DNS, and vhost enumeration. Use the `-x` flag to add file extensions.

| Mode | Wordlist | Path |
|------|----------|------|
| Dir (fast) | `common.txt` | `$SL/Discovery/Web-Content/common.txt` |
| Dir (standard) | `directory-list-2.3-medium.txt` | `$SL/Discovery/Web-Content/` |
| Dir (thorough) | `raft-large-directories.txt` | `$SL/Discovery/Web-Content/` |
| DNS subdomain | `subdomains-top1million-5000.txt` | `$SL/Discovery/DNS/` |
| Vhost | `subdomains-top1million-5000.txt` | `$SL/Discovery/DNS/` |

```bash
# Directory scan — standard
gobuster dir -u http://10.10.10.10 \
             -w $SL/Discovery/Web-Content/directory-list-2.3-medium.txt \
             -x php,html,txt,bak -t 50

# Directory scan — add PHP extension list
gobuster dir -u http://10.10.10.10 \
             -w $SL/Discovery/Web-Content/common.txt \
             -w $SL/Discovery/Web-Content/Common-PHP-Filenames.txt

# DNS subdomain brute-force
gobuster dns -d target.com \
             -w $SL/Discovery/DNS/subdomains-top1million-5000.txt \
             -t 50

# Vhost enumeration
gobuster vhost -u http://target.com \
               -w $SL/Discovery/DNS/subdomains-top1million-5000.txt \
               --append-domain
```

---

### ffuf

ffuf replaces the `FUZZ` keyword in any part of the request — URL path, header, host header, POST body.

| Mode | Wordlist | Path |
|------|----------|------|
| Dir (fast) | `common.txt` | `$SL/Discovery/Web-Content/common.txt` |
| Dir (standard) | `directory-list-2.3-medium.txt` | `$SL/Discovery/Web-Content/` |
| Dir (thorough) | `raft-large-files.txt` | `$SL/Discovery/Web-Content/` |
| Subdomain/vhost | `subdomains-top1million-5000.txt` | `$SL/Discovery/DNS/` |
| API routes | Assetnote `httparchive_apiroutes_*.txt` | https://wordlists.assetnote.io/ |
| Parameter names | `burp-parameter-names.txt` | `$SL/Discovery/Web-Content/` |

```bash
# Web content discovery
ffuf -w $SL/Discovery/Web-Content/directory-list-2.3-medium.txt \
     -u http://10.10.10.10/FUZZ \
     -mc 200,301,302,403 -t 50

# Vhost fuzzing (filter by response size to remove baseline)
ffuf -w $SL/Discovery/DNS/subdomains-top1million-5000.txt \
     -u http://10.10.10.10 \
     -H "Host: FUZZ.target.com" \
     -fs 4242   # filter out baseline response size

# File extension brute-force
ffuf -w $SL/Discovery/Web-Content/common.txt \
     -u http://10.10.10.10/FUZZ \
     -e .php,.html,.txt,.bak,.zip,.tar.gz

# POST parameter fuzzing
ffuf -w $SL/Fuzzing/LFI/LFI-Jhaddix.txt \
     -u http://10.10.10.10/page.php \
     -X POST -d "file=FUZZ" \
     -H "Content-Type: application/x-www-form-urlencoded"
```

---

### Feroxbuster

Feroxbuster is a recursive content discovery tool — it re-runs on every discovered directory.

```bash
# Recursive directory scan
feroxbuster -u http://10.10.10.10 \
            -w $SL/Discovery/Web-Content/raft-large-directories.txt \
            -x php,html,txt -t 50 --depth 3

# With auto-tune and filter
feroxbuster -u http://10.10.10.10 \
            -w $SL/Discovery/Web-Content/directory-list-2.3-medium.txt \
            --filter-status 404,403
```

---

### dnsenum / fierce / dnsrecon

DNS enumeration tools for subdomain brute-forcing.

| Scenario | Wordlist |
|----------|----------|
| Fast (CTF/quick) | `$SL/Discovery/DNS/subdomains-top1million-5000.txt` |
| Standard engagement | `$SL/Discovery/DNS/subdomains-top1million-20000.txt` |
| Thorough | `$SL/Discovery/DNS/subdomains-top1million-110000.txt` |
| Bitmasked/alt permutations | `$SL/Discovery/DNS/dns-Jhaddix.txt` |

```bash
# dnsenum
dnsenum --dnsserver 10.10.10.10 --enum -p 0 -s 0 \
        -f $SL/Discovery/DNS/subdomains-top1million-5000.txt \
        target.com

# dnsrecon — brute-force subdomains
dnsrecon -d target.com -D $SL/Discovery/DNS/subdomains-top1million-5000.txt -t brt

# fierce
fierce --domain target.com \
       --wordlist $SL/Discovery/DNS/subdomains-top1million-5000.txt
```

---

### onesixtyone (SNMP)

onesixtyone only needs a community string list.

| Use Case | Wordlist |
|----------|---------|
| Default community strings | `$SL/Discovery/SNMP/common-snmp-community-strings.txt` |
| Extended SNMP brute-force | `$SL/Discovery/SNMP/snmp-onesixtyone.txt` |

```bash
onesixtyone -c $SL/Discovery/SNMP/common-snmp-community-strings.txt \
            -i targets.txt -o snmp_results.txt

# Single host
onesixtyone -c $SL/Discovery/SNMP/common-snmp-community-strings.txt \
            10.10.10.10
```

---

### Hashcat / John the Ripper

Password cracking tools — input is a hash file, wordlist is the candidate password source.

| Attack Type | Wordlist | Rule File |
|-------------|----------|-----------|
| General cracking | `$WL/rockyou.txt` | `best64.rule` |
| Fast common passwords | `$SL/Passwords/Common-Credentials/10-million-password-list-top-1000.txt` | none |
| Targeted (org-specific) | CeWL output | `best64.rule` |
| NTLM hashes | `$WL/rockyou.txt` | `OneRuleToRuleThemAll.rule` — [github.com/NotSoSecure/password_cracking_rules](https://github.com/NotSoSecure/password_cracking_rules) (not installed by default) |
| WPA/WPA2 handshakes | `$WL/rockyou.txt` | `best64.rule` |

```bash
# Hashcat — NTLM with rockyou + best64 rules
hashcat -m 1000 hashes.txt $WL/rockyou.txt \
        -r /usr/share/hashcat/rules/best64.rule

# Hashcat — MD5 with top-1000
hashcat -m 0 hashes.txt \
        $SL/Passwords/Common-Credentials/10-million-password-list-top-1000.txt

# Hashcat — IPMI RAKP (mode 7300)
hashcat -m 7300 ipmi_hashes.txt $WL/rockyou.txt

# Hashcat — WPA2 handshake
hashcat -m 2500 handshake.hccapx $WL/rockyou.txt

# John the Ripper — general
john --wordlist=$WL/rockyou.txt hashes.txt

# John with rules
john --wordlist=$WL/rockyou.txt --rules=best64 hashes.txt

# Available Hashcat rules (built-in)
ls /usr/share/hashcat/rules/

# Download OneRuleToRuleThemAll (not pre-installed)
wget -O /usr/share/hashcat/rules/OneRuleToRuleThemAll.rule \
     https://raw.githubusercontent.com/NotSoSecure/password_cracking_rules/master/OneRuleToRuleThemAll.rule
```

---

### Metasploit Auxiliary Modules

Metasploit's brute-force modules use built-in or user-supplied wordlists.

| Module | Username List | Password List |
|--------|--------------|---------------|
| `auxiliary/scanner/ftp/ftp_login` | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| `auxiliary/scanner/ssh/ssh_login` | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| `auxiliary/scanner/smb/smb_login` | `$SL/Usernames/top-usernames-shortlist.txt` | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |
| `auxiliary/scanner/mssql/mssql_login` | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| `auxiliary/scanner/mysql/mysql_login` | `$SL/Usernames/top-usernames-shortlist.txt` | `$WL/rockyou.txt` |
| `auxiliary/scanner/vnc/vnc_login` | *(not applicable)* | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` |

```bash
msf6> use auxiliary/scanner/ssh/ssh_login
msf6> set RHOSTS 10.10.10.10
msf6> set USER_FILE /usr/share/seclists/Usernames/top-usernames-shortlist.txt
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set THREADS 5
msf6> run
```

---

### Nmap NSE (brute scripts)

Nmap has built-in brute-force scripts for many services. They default to small internal lists but accept custom ones.

```bash
# FTP brute with custom lists
nmap -p21 --script ftp-brute \
     --script-args userdb=$SL/Usernames/top-usernames-shortlist.txt,\
                   passdb=$WL/rockyou.txt \
     10.10.10.10

# SSH brute
nmap -p22 --script ssh-brute \
     --script-args userdb=$SL/Usernames/top-usernames-shortlist.txt,\
                   passdb=$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
     10.10.10.10

# HTTP brute
nmap -p80 --script http-brute \
     --script-args userdb=$SL/Usernames/top-usernames-shortlist.txt,\
                   passdb=$WL/rockyou.txt \
     10.10.10.10
```

---

### WPScan (WordPress)

WPScan enumerates WordPress users, plugins, and themes, and can brute-force passwords.

| Use Case | Wordlist |
|----------|---------|
| User password brute-force | `$WL/rockyou.txt` |
| Plugin enumeration | Built-in WPScan DB (no wordlist needed) |

```bash
# Enumerate users + brute-force passwords
wpscan --url http://target.com \
       --enumerate u \
       --passwords $WL/rockyou.txt

# Password attack against known user
wpscan --url http://target.com \
       -U admin \
       -P $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt
```

---

## By Service

### FTP (port 21)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | Try `anonymous` first manually |
| Passwords | `$WL/rockyou.txt` | 14M passwords |
| Default creds | `$SL/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt` | Try before brute-force |

### SSH (port 22)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | SSH is slow to brute — use small lists |
| Passwords | `$WL/rockyou.txt` | Rate-limit with `-t 4` in Hydra |
| Default creds | `$SL/Passwords/Default-Credentials/ssh-betterdefaultpasslist.txt` | |

### SMTP (port 25/587)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| User enumeration (VRFY) | `$SL/Usernames/top-usernames-shortlist.txt` | Use smtp-user-enum with -U flag |
| User enumeration (large) | `$SL/Usernames/xato-net-10-million-usernames.txt` | Slow; use on permissive servers |
| Auth brute-force | `$WL/rockyou.txt` | Only if AUTH supported |

```bash
# SMTP user enumeration with smtp-user-enum
smtp-user-enum -M VRFY \
               -U $SL/Usernames/top-usernames-shortlist.txt \
               -t 10.10.10.10
```

### SMB (port 139/445)

| Wordlist Need | File | Tool | Notes |
|--------------|------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | CME / MSF | Or enumerate first via RID brute |
| Passwords — spray | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` | **NetExec** | Short list, many hosts/users — CME is ideal |
| Passwords — brute-force | `$WL/rockyou.txt` | **MSF smb_login** | Large wordlist against one account — use MSF, not CME |
| Default creds | `$SL/Passwords/Default-Credentials/` | CME / Hydra | Try before brute-force |

**Tool selection for SMB passwords:**
- **NetExec** (`-p shortlist.txt`): designed for spraying a small list across many hosts. Fast, multi-threaded, stores results. Unreliable with large files like rockyou.txt.
- **MSF `smb_login`**: designed for brute-force — handles rockyou.txt reliably, `STOP_ON_SUCCESS` halts immediately on a hit, thread count is controllable.

> **If CME with rockyou.txt returned no results:** first confirm the file is unzipped (`ls -lh /usr/share/wordlists/rockyou.txt` — should be ~134 MB, not ~51 MB). The `.gz` file passes binary data as passwords and will always fail silently.

```bash
# SMB brute-force — correct approach with MSF
msf6> use auxiliary/scanner/smb/smb_login
msf6> set RHOSTS 10.10.10.10
msf6> set SMBUser jason
msf6> set PASS_FILE /usr/share/wordlists/rockyou.txt
msf6> set STOP_ON_SUCCESS true
msf6> set THREADS 3
msf6> set VERBOSE false
msf6> run

# SMB password spray — correct approach with CME
nxc smb 10.10.10.0/24 \
             -u $SL/Usernames/top-usernames-shortlist.txt \
             -p $SL/Passwords/Common-Credentials/top-passwords-shortlist.txt \
             --no-bruteforce --continue-on-success

# CME — check password policy BEFORE spraying
nxc smb 10.10.10.10 -u '' -p '' --pass-pol
```

### SNMP (UDP 161)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Community strings | `$SL/Discovery/SNMP/common-snmp-community-strings.txt` | ~120 strings |
| Extended brute-force | `$SL/Discovery/SNMP/snmp-onesixtyone.txt` | Larger list |

### HTTP/HTTPS (port 80/443)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Directory scan | `$SL/Discovery/Web-Content/directory-list-2.3-medium.txt` | Standard engagement default |
| Backup/config files | `$SL/Discovery/Web-Content/raft-large-files.txt` | Catches .bak, .old, .tar.gz |
| PHP files | `$SL/Discovery/Web-Content/Common-PHP-Filenames.txt` | When PHP detected |
| Subdomains | `$SL/Discovery/DNS/subdomains-top1million-5000.txt` | Vhost + DNS |
| API routes | Assetnote httparchive lists | https://github.com/assetnote/wordlists (see `data/` directory) |
| Login brute | `$WL/rockyou.txt` + `$SL/Usernames/top-usernames-shortlist.txt` | Via ffuf or Hydra |

### MySQL (port 3306) / MSSQL (port 1433)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | `root`, `sa`, `admin` most important |
| Passwords | `$WL/rockyou.txt` | Try blank password manually first |
| Default creds | `$SL/Passwords/Default-Credentials/` | Check for service-specific files |

### RDP (port 3389)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | `Administrator`, `admin` first |
| Passwords (spray) | `$SL/Passwords/Common-Credentials/top-passwords-shortlist.txt` | Lockout risk is high on RDP |

### IPMI (UDP 623)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | `ADMIN`, `root`, `Administrator` |
| Passwords (default) | `$SL/Passwords/Default-Credentials/` | Dell: `calvin`; Supermicro: `ADMIN` |
| Hash cracking | `$WL/rockyou.txt` | After RAKP dump — Hashcat mode 7300 |

### Oracle TNS (port 1521)

| Wordlist Need | File | Notes |
|--------------|------|-------|
| SID names | `$SL/Passwords/Default-Credentials/oracle-default-passwords.csv` | Or use ODAT sidguesser |
| Usernames | `$SL/Usernames/top-usernames-shortlist.txt` | `scott`, `dbsnmp`, `system`, `sys` |
| Passwords | `$WL/rockyou.txt` | Default: `tiger`, `oracle`, `manager` |

---

## Default Credential Lists

Always try default credentials before starting a brute-force. These lists are small and run in seconds.

| Service / Device | File | Location |
|-----------------|------|----------|
| FTP | `ftp-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| SSH | `ssh-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| Telnet | `telnet-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| HTTP (various devices) | `default-passwords.csv` | `$SL/Passwords/Default-Credentials/` |
| MySQL | `mysql-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| Postgres | `postgres-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| Oracle | `oracle-default-passwords.csv` | `$SL/Passwords/Default-Credentials/` |
| Tomcat | `tomcat-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| Routers / Switches | `router-betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |
| Combined (all services) | `betterdefaultpasslist.txt` | `$SL/Passwords/Default-Credentials/` |

```bash
# List all available default credential files
ls /usr/share/seclists/Passwords/Default-Credentials/

# Test FTP defaults with Hydra
hydra -C $SL/Passwords/Default-Credentials/ftp-betterdefaultpasslist.txt \
      ftp://10.10.10.10
# -C = colon-separated user:pass file
```

---

## Related Pages

- [[wordlists/_overview]] — installation paths, sources, custom generation
- [[tools/attack/hydra]]
- [[tools/gobuster]]
- [[tools/ffuf]]
- [[tools/attack/medusa]]
- [[tools/attack/netexec]]
- [[enumeration/snmp]]
- [[enumeration/dns]]
- [[enumeration/ftp]]
- [[enumeration/smb]]
