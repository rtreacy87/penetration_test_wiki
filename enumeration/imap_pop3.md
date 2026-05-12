---
tags: [enumeration, enumeration/imap_pop3]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# IMAP / POP3 Enumeration

IMAP and POP3 enumeration: banner grabbing, command-line interaction, OpenSSL TLS sessions, and authentication probing.

## Overview

IMAP (Internet Message Access Protocol) and POP3 (Post Office Protocol 3) are the protocols for retrieving email from a mail server. They complement SMTP (which handles sending).

- **IMAP**: Port 143 (plaintext) / Port 993 (TLS). Full mailbox management — folder structures, multi-client sync, server-side storage.
- **POP3**: Port 110 (plaintext) / Port 995 (TLS). Simple fetch-and-delete model — no folder support, emails downloaded locally.

If credentials are obtained during an engagement, these services provide access to employee email — a high-value finding that can reveal business processes, credentials shared via email, and sensitive communications.

## Key Concepts / Techniques

### IMAP vs POP3

| Feature | IMAP | POP3 |
|---------|------|------|
| Folder structures | Yes | No |
| Server-side storage | Yes | No (downloads locally) |
| Multi-client sync | Yes | No |
| Offline access | Limited | Yes (downloads) |
| Port (plaintext) | 143 | 110 |
| Port (TLS) | 993 | 995 |

### What to Look For

- **Banner**: reveals software (Dovecot, Cyrus, Exchange) and version
- **Capabilities**: what authentication methods are supported (AUTH=PLAIN, STARTTLS, etc.)
- **Certificate details**: hostname, organization, email (from SSL cert)
- **Email content**: if credentials are obtained — configs, passwords, internal comms

### Dangerous Configuration Settings (Dovecot)

| Setting | Description |
|---------|-------------|
| `auth_debug` | Enable all authentication debug logging |
| `auth_debug_passwords` | Log submitted passwords |
| `auth_verbose` | Log failed auth attempts and reasons |
| `auth_verbose_passwords` | Log passwords used for auth |
| `auth_anonymous_username` | Username for anonymous SASL access |

## Commands / Syntax

```bash
# Nmap scan for all IMAP/POP3 ports
sudo nmap 10.129.14.128 -sV -p110,143,993,995 -sC

# IMAP via curl (lists mailboxes)
curl -k 'imaps://10.129.14.128' --user user:p4ssw0rd

# IMAP via curl with verbose (shows TLS info, capabilities, folders)
curl -k 'imaps://10.129.14.128' --user cry0l1t3:1234 -v

# OpenSSL — TLS interaction with IMAP (port 993)
openssl s_client -connect 10.129.14.128:imaps

# OpenSSL — TLS interaction with POP3 (port 995)
openssl s_client -connect 10.129.14.128:pop3s

# Manual IMAP session (after OpenSSL connects)
1 LOGIN cry0l1t3 1234
1 LIST "" *
1 SELECT INBOX
1 FETCH 1 all
1 FETCH 1 BODY[]
1 LOGOUT

# Manual POP3 session (after OpenSSL connects)
USER cry0l1t3
PASS 1234
STAT
LIST
RETR 1
QUIT
```

## Flags & Options

### IMAP Commands

| Command | Description |
|---------|-------------|
| `1 LOGIN <user> <pass>` | Authenticate |
| `1 LIST "" *` | List all directories/folders |
| `1 SELECT INBOX` | Open a mailbox |
| `1 FETCH <id> all` | Retrieve message metadata |
| `1 FETCH <id> BODY[]` | Retrieve full message body |
| `1 LSUB "" *` | List subscribed folders |
| `1 CLOSE` | Remove messages flagged for deletion |
| `1 LOGOUT` | End session |

### POP3 Commands

| Command | Description |
|---------|-------------|
| `USER <username>` | Identify user |
| `PASS <password>` | Authenticate |
| `STAT` | Count and size of messages |
| `LIST` | List all messages with sizes |
| `RETR <id>` | Download specific message |
| `DELE <id>` | Delete specific message |
| `CAPA` | Show server capabilities |
| `QUIT` | End session |

## Version Detection & Exploit Research

IMAP/POP3 banners expose the server software (Dovecot, Cyrus, Exchange) and version at connection time. The TLS certificate provides additional infrastructure intelligence. These services have had critical pre-authentication RCE vulnerabilities, making version identification essential before moving to authenticated enumeration.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap scan | `nmap -sV -p110,143,993,995 -sC <IP>` | Software, version, capabilities |
| IMAP banner (plaintext) | `nc -nv <IP> 143` | `* OK Dovecot ready` — software + version |
| POP3 banner (plaintext) | `nc -nv <IP> 110` | `+OK Dovecot ready` |
| IMAP TLS banner | `openssl s_client -connect <IP>:imaps` | Version string in OK line + cert details |
| POP3 TLS banner | `openssl s_client -connect <IP>:pop3s` | Version string + cert details |
| IMAP CAPABILITY | Send `1 CAPABILITY` after connecting | Auth methods, extensions, implementation hints |

### Searching for Exploits

```bash
# Searchsploit
searchsploit dovecot
searchsploit cyrus imap
searchsploit exchange imap

# Metasploit
msf6> search type:exploit name:dovecot
msf6> search type:exploit name:imap
```

### Notable CVEs

| CVE | Software / Version | Impact |
|-----|-------------------|--------|
| CVE-2019-11500 | Dovecot < 2.3.7.2 | Pre-auth heap buffer overflow via IMAP/ManageSieve — RCE |
| CVE-2017-15132 | Dovecot < 2.2.34 | Memory leak in login process — info disclosure |
| CVE-2019-18928 | Cyrus IMAP < 2.5.15 | HTTP/2 via IMAP PROXY — auth bypass |
| CVE-2021-26855 | Exchange Server 2013–2019 | SSRF pre-auth (ProxyLogon chain) — RCE |
| CVE-2021-27065 | Exchange Server 2013–2019 | Post-auth file write (ProxyLogon chain) — RCE |

## Gotchas & Notes

- **SSL certificates reveal infrastructure**: The common name (CN) and Subject Alternative Names in the certificate expose hostnames, email addresses, and organizational structure. Always capture this.
- **Capability listing reveals authentication methods**: If `AUTH=PLAIN` is available without TLS enforcement, credentials traverse the network in base64 (easily decoded).
- **Credential reuse**: Email credentials are frequently reused elsewhere. Finding IMAP/POP3 credentials opens doors to SSH, VPN, or web applications.
- **Context note from lab**: User `robin` was found via SMTP VRFY, and their password was the same as their username (`robin:robin`). Always try username-as-password for mail accounts.
- **Dovecot is the most common Linux IMAP/POP3 server**. Its debugging settings can inadvertently log credentials in plaintext.
- **Port 143 without STARTTLS** = credentials sent in plaintext. This is a separate finding (cleartext authentication).
- The `FETCH` command can retrieve individual emails by ID number. If you gain access, do not just list folders — iterate through recent emails for credential-containing content.

## Related Pages

- [[enumeration/_overview]]
- [[enumeration/smtp]]
- [[labs/htb/attacking_common_services/medium_skill_assessment]] — POP3 brute-force with targeted wordlist → email contains SSH private key → shell access

## Sources

- raw/footprinting/host_based_enumeration_imap_pop3.md
