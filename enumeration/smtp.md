---
tags: [enumeration, enumeration/smtp]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# SMTP Enumeration

SMTP enumeration: VRFY/EXPN user enumeration, open relay detection, banner grabbing, and nmap scripts.

## Overview

SMTP (Simple Mail Transfer Protocol) handles email transmission. Default port is **25** (server-to-server); port **587** is for authenticated client submission (often with STARTTLS); port **465** is for SMTP over SSL.

Two key attack surfaces:
1. **User enumeration** via VRFY/EXPN commands
2. **Open relay** — server accepts mail for any destination, enabling spam and spoofing

SMTP transmits in cleartext by default. With ESMTP (Extended SMTP), STARTTLS can upgrade the connection.

## Key Concepts / Techniques

### Email Infrastructure

The email delivery chain:
`Client (MUA)` → `Submission Agent (MSA)` → `Open Relay (MTA)` → `Mail Delivery Agent (MDA)` → `Mailbox (POP3/IMAP)`

- **MUA**: Mail User Agent (email client)
- **MSA**: Mail Submission Agent (validates origin)
- **MTA**: Mail Transfer Agent (routes mail)
- **MDA**: Mail Delivery Agent (delivers to mailbox)

An **open relay** is an MTA that forwards mail for any sender to any recipient without authentication.

### SMTP Commands for Enumeration

| Command | Description |
|---------|-------------|
| `HELO / EHLO` | Initiate session; EHLO lists server extensions |
| `VRFY <user>` | Verify if mailbox exists |
| `EXPN <list>` | Expand a mailing list (reveals members) |
| `MAIL FROM:` | Specify sender |
| `RCPT TO:` | Specify recipient |
| `DATA` | Begin message body |
| `RSET` | Reset session |
| `NOOP` | Keep-alive |
| `QUIT` | End session |
| `AUTH PLAIN` | Authenticate |

### VRFY Caveat

VRFY may return code `252` for any input regardless of whether the user exists — server response depends on configuration. Do not treat `252` as definitive confirmation. Verify with other methods.

### Open Relay

If `mynetworks = 0.0.0.0/0` in Postfix config (or equivalent in other MTAs), the server relays mail for anyone. Test with the `smtp-open-relay` NSE script.

## Commands / Syntax

```bash
# Nmap scan with SMTP scripts
sudo nmap 10.129.14.128 -sC -sV -p25

# Open relay test (16 different tests)
sudo nmap 10.129.14.128 -p25 --script smtp-open-relay -v

# Manual SMTP interaction via telnet
telnet 10.129.14.128 25

# Once connected:
HELO mail1.inlanefreight.htb          # Basic greeting
EHLO mail1                             # Extended greeting — lists capabilities

VRFY root                              # Check if root mailbox exists
VRFY cry0l1t3                          # Check another user

# Send a test email
EHLO inlanefreight.htb
MAIL FROM: <attacker@attacker.com>
RCPT TO: <target@inlanefreight.htb> NOTIFY=success,failure
DATA
From: <attacker@attacker.com>
To: <target@inlanefreight.htb>
Subject: Test
Body of the email.
.                                       # End of message (dot on its own line)
QUIT

# Via web proxy
CONNECT 10.129.14.128:25 HTTP/1.0
```

## Flags & Options

### Nmap SMTP Scripts

| Script | Description |
|--------|-------------|
| `smtp-commands` | Lists all commands via EHLO |
| `smtp-open-relay` | Tests 16 open relay configurations |
| `smtp-enum-users` | Attempts user enumeration via VRFY/EXPN/RCPT |
| `smtp-brute` | Credential brute-force |
| `smtp-ntlm-info` | Retrieves NTLM server info |

### SMTP Response Codes

| Code | Meaning |
|------|---------|
| 220 | Service ready (banner) |
| 250 | Action completed successfully |
| 252 | Cannot verify user, but will forward (unreliable for user enum) |
| 354 | Start mail input; end with `<CRLF>.<CRLF>` |
| 421 | Service not available |
| 530 | Authentication required |
| 550 | User does not exist |

## Version Detection & Exploit Research

The SMTP 220 banner is transmitted immediately upon connection and almost universally identifies the MTA software and version. This is pre-authentication, requiring no credentials. The MTA software (Postfix, Sendmail, Exim, Exchange, OpenSMTPD) determines the CVE search space — each has its own vulnerability history and the version string narrows it to specific bugs.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Banner grab (netcat) | `nc -nv <IP> 25` | `220 mail.example.com ESMTP Postfix (Ubuntu)` |
| Banner grab (telnet) | `telnet <IP> 25` | Full 220 banner with software + version |
| Nmap smtp-commands | `nmap -p25 --script smtp-commands <IP>` | EHLO response — capabilities + server name |
| Nmap smtp-ntlm-info | `nmap -p25 --script smtp-ntlm-info <IP>` | Windows hostname + domain (Exchange) |
| EHLO command | Connect, then send `EHLO test` | Extensions list reveals auth methods, size limits, STARTTLS |
| Nmap service scan | `nmap -sV -p25,465,587 <IP>` | Full version string from service probe |

**Common banner formats:**
- `220 mail.corp.com ESMTP Postfix` → Postfix (version rarely in banner)
- `220 mx.corp.com ESMTP Sendmail 8.15.2/8.15.2` → Sendmail with version
- `220 mail.corp.com ESMTP Exim 4.95 ...` → Exim with version (critical to note)
- `220 CORP-EXCH01 Microsoft ESMTP MAIL Service` → Exchange

### Searching for Exploits

```bash
# Searchsploit
searchsploit postfix
searchsploit exim
searchsploit sendmail
searchsploit opensmtpd
searchsploit "exchange smtp"

# Metasploit
msf6> search type:exploit name:smtp
msf6> search type:exploit name:exim
msf6> search cve:2020-7247
```

### Notable CVEs

| CVE | Software / Version | Impact |
|-----|-------------------|--------|
| CVE-2020-7247 | OpenSMTPD < 6.6.2 | Unauthenticated RCE as root via malformed MAIL FROM |
| CVE-2019-10149 | Exim < 4.92.1 | Unauthenticated RCE via RCPT TO address parsing (The Return of the WIZard) |
| CVE-2021-27928 | Exim < 4.94.2 | Remote code execution via SPA/NTLM auth (auth required) |
| CVE-2014-3566 | Sendmail + SSLv3 | POODLE — decrypt TLS via SSLv3 fallback |
| CVE-2021-26855 | Exchange 2013–2019 | SSRF pre-auth (ProxyLogon) — leads to arbitrary code execution |

## Gotchas & Notes

- **VRFY returning 252 for every input** is a common misconfiguration trap. The server is configured to acknowledge without verifying. Look for `550` (definitively doesn't exist) or `250` (definitively exists) for reliable results.
- The **EHLO response** reveals all server capabilities: PIPELINING, SIZE limits, AUTH methods, STARTTLS support. This determines what is possible on the server.
- **Open relay** is a critical finding. It enables sending email from any sender to any recipient through the server — useful for phishing and spoofing attacks.
- **Banner information**: SMTP banners often reveal the server software (Postfix, Sendmail, Exchange). Document the version.
- SMTP **does not authenticate senders by default** — mail spoofing is trivial on an open relay.
- SPF, DKIM, and DMARC records (found in DNS TXT records) are the defenses against open relay abuse. If these exist but the server is still an open relay, those records just provide false assurance.

## Related Pages

- [[attack/smtp]] — exploitation: user enum, O365spray, open relay abuse, CVE-2020-7247 OpenSMTPD RCE
- [[tools/enumeration/smtp_user_enum]] — automated VRFY/EXPN/RCPT enumeration with wordlists
- [[enumeration/_overview]]
- [[enumeration/imap_pop3]]
- [[enumeration/dns]]
- [[tools/attack/hydra]]

## Sources

- raw/footprinting/host_based_enumeration_smtp.md
