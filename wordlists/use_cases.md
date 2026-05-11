---
tags: [wordlist]
module: multi
last_updated: 2026-05-10
source_count: 0
---

# Wordlist Use Cases

Task-to-wordlist mapping for common penetration testing scenarios.

## Overview

Choosing the right wordlist is a trade-off between coverage (larger lists) and speed (smaller lists). Start small and escalate.

## By target service

### Web directories (gobuster / ffuf / feroxbuster)

| Scenario | Wordlist | Size | Notes |
|----------|----------|------|-------|
| Quick check | `Discovery/Web-Content/common.txt` | ~4K | Fast first pass |
| Standard scan | `Discovery/Web-Content/directory-list-2.3-medium.txt` | ~220K | Default for most engagements |
| Thorough | `Discovery/Web-Content/raft-large-directories.txt` | ~330K | Includes tricky entries |
| PHP-specific | `Discovery/Web-Content/Common-PHP-Filenames.txt` | ~3K | Add when PHP detected |
| Backup files | `Discovery/Web-Content/raft-large-files.txt` | ~450K | Finds .bak, .old, .orig |

### DNS subdomain brute-force

| Scenario | Wordlist | Notes |
|----------|----------|-------|
| Fast | `Discovery/DNS/subdomains-top1million-5000.txt` | Top 5K — fast first pass |
| Standard | `Discovery/DNS/subdomains-top1million-20000.txt` | Covers most environments |
| Thorough | `Discovery/DNS/subdomains-top1million-110000.txt` | Large, slow |

### SNMP community strings

| Wordlist | Notes |
|----------|-------|
| `Discovery/SNMP/common-snmp-community-strings.txt` | ~120 strings; try first |
| `Discovery/SNMP/snmp-onesixtyone.txt` | Onesixtyone-formatted |

### Username enumeration

| Scenario | Wordlist | Notes |
|----------|----------|-------|
| Generic | `Usernames/xato-net-10-million-usernames-dup.txt` | Large; good for SMTP VRFY |
| Short | `Usernames/top-usernames-shortlist.txt` | ~20 common usernames |
| Custom | Generated with username-anarchy from employee list | Best for targeted orgs |

### Password attacks

| Scenario | Wordlist | Notes |
|----------|----------|-------|
| Password spray | `Passwords/Common-Credentials/top-passwords-shortlist.txt` | Use for spraying — avoid lockout |
| General brute-force | `Passwords/Leaked-Databases/rockyou.txt` | 14M passwords; standard |
| Fast crack | `Passwords/Common-Credentials/10-million-password-list-top-100.txt` | 100 passwords — quick wins |
| Targeted | CeWL output from target site | Best for custom password policies |

## By tool

### ffuf

```bash
# Web content discovery
ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
     -u http://target.com/FUZZ -mc 200,301,302,403

# Subdomain fuzzing
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u http://FUZZ.target.com -mc 200,301,302
```

### gobuster

```bash
# Directory mode
gobuster dir -u http://target.com \
             -w /usr/share/seclists/Discovery/Web-Content/common.txt \
             -x php,html,txt

# DNS mode
gobuster dns -d target.com \
             -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### hydra

```bash
# SSH brute-force
hydra -l admin -P /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt \
      ssh://target.com

# HTTP POST form
hydra -l admin -P rockyou.txt target.com http-post-form \
      "/login:username=^USER^&password=^PASS^:Invalid"
```

### onesixtyone (SNMP)

```bash
onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt \
            -i targets.txt
```

## Related pages

- [[wordlists/_overview]]
- [[tools/nmap]]
- [[enumeration/snmp]]
- [[enumeration/dns]]
