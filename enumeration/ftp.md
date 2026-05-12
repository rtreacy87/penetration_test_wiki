---
tags: [enumeration, enumeration/ftp]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# FTP Enumeration

FTP enumeration: anonymous login, banner grabbing, nmap scripts, recursive listing, file download/upload, and TFTP coverage.

## Overview

FTP (File Transfer Protocol) is one of the oldest protocols on the internet and remains commonly misconfigured. It uses TCP port 21 for the control channel and TCP port 20 for data transfer. FTP transmits everything in cleartext — credentials, commands, and data — making it sniffable on the wire.

The most critical misconfiguration: **anonymous access**. Many FTP servers are configured to allow anonymous login, often with read (and sometimes write) permissions on directories containing sensitive files.

## Key Concepts / Techniques

### Active vs. Passive Mode

- **Active mode**: Client opens a random port; server connects back to it. Blocked by most firewalls protecting the client.
- **Passive mode**: Server opens a random port; client connects to it. Firewalls pass this because the client initiates.

In practice, always use passive mode when scanning from outside a network.

### Anonymous Login

When `anonymous_enable=YES` is set (vsFTPd config), any user can log in with username `anonymous` and any (or no) password. This is a finding on its own, even if no sensitive files are exposed.

### Key vsFTPd Settings (Dangerous)

| Setting | Description |
|---------|-------------|
| `anonymous_enable=YES` | Allow anonymous login |
| `anon_upload_enable=YES` | Allow anonymous file upload |
| `anon_mkdir_write_enable=YES` | Allow anonymous directory creation |
| `no_anon_password=YES` | No password required for anonymous |
| `write_enable=YES` | Allow STOR, DELE, RNFR, RNTO, MKD, RMD, APPE |
| `hide_ids=YES` | Show 'ftp' instead of real UID/GID — hides user info |
| `ls_recurse_enable=YES` | Allow recursive directory listing |

### TFTP

Trivial FTP uses **UDP** (not TCP), has no authentication, and is restricted to files with world-readable/writable permissions. Found in legacy environments and IoT/PXE setups. Does not support directory listing.

## Commands / Syntax

```bash
# Nmap service scan with default scripts
sudo nmap -sV -p21 -sC -A 10.129.14.136

# Nmap with script trace (shows NSE communication)
sudo nmap -sV -p21 -sC -A 10.129.14.136 --script-trace

# Find all FTP-related NSE scripts
find / -type f -name ftp* 2>/dev/null | grep scripts/

# Update NSE script database
sudo nmap --script-updatedb

# Connect with the FTP client
ftp 10.129.14.136
# Login as anonymous
# Username: anonymous, Password: (blank or any string)

# FTP client commands
ftp> status          # Show connection details
ftp> debug           # Enable debug output
ftp> trace           # Show packet trace
ftp> ls              # List files
ftp> ls -R           # Recursive listing (if ls_recurse_enable=YES)
ftp> get "Important Notes.txt"    # Download file
ftp> put testupload.txt           # Upload file

# Download all accessible files (active mode disabled)
wget -m --no-passive ftp://anonymous:anonymous@10.129.14.136

# Banner grab with netcat
nc -nv 10.129.14.136 21

# Banner grab with telnet
telnet 10.129.14.136 21

# FTP over TLS/SSL (reveals certificate details)
openssl s_client -connect 10.129.14.136:21 -starttls ftp
```

## Flags & Options

### Nmap FTP NSE Scripts

| Script | Description |
|--------|-------------|
| `ftp-anon` | Checks for anonymous login; lists root directory if successful |
| `ftp-syst` | Executes STAT command, reveals FTP server status and version |
| `ftp-vsftpd-backdoor` | Tests for vsFTPd 2.3.4 backdoor (CVE-2011-2523) |
| `ftp-brute` | Credential brute-force |
| `ftp-bounce` | Tests for FTP bounce attack vector |
| `ftp-proftpd-backdoor` | Tests for ProFTPD backdoor |
| `ftp-vuln-cve2010-4221` | Tests for ProFTPD buffer overflow |

### FTP Status Codes

| Code | Meaning |
|------|---------|
| 220 | Service ready (banner) |
| 230 | Login successful |
| 331 | Username OK, password required |
| 530 | Login failed |
| 425 | Can't open data connection |
| 226 | Transfer complete |

## Version Detection & Exploit Research

The FTP 220 banner is the most reliable pre-authentication version source — it almost always identifies the server software and version. Grab it with netcat before attempting any other enumeration. Historical FTP implementations (particularly vsFTPd and ProFTPD) have serious RCE vulnerabilities tied to specific version strings.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Banner grab (netcat) | `nc -nv <IP> 21` | `220 (vsFTPd 3.0.3)` or similar |
| Banner grab (telnet) | `telnet <IP> 21` | Full 220 banner |
| Nmap service scan | `nmap -sV -p21 <IP>` | Software + version |
| NSE ftp-syst | `nmap -p21 --script ftp-syst <IP>` | STAT output — OS and FTP server info |
| TLS cert (FTPS) | `openssl s_client -connect <IP>:21 -starttls ftp` | Certificate fields, server info |

### Searching for Exploits

```bash
# Searchsploit
searchsploit vsftpd
searchsploit proftpd
searchsploit filezilla server

# Metasploit
msf6> search type:exploit name:vsftpd
msf6> search type:exploit name:proftpd
msf6> search cve:2011-2523
```

### Notable CVEs

| CVE | Software / Version | Impact |
|-----|-------------------|--------|
| CVE-2011-2523 | vsFTPd 2.3.4 | Backdoor — BIND shell on port 6200 via `:)` in username |
| CVE-2010-4221 | ProFTPD < 1.3.3c | Stack buffer overflow — unauthenticated RCE |
| CVE-2015-3306 | ProFTPD < 1.3.5b | mod_copy RCE — arbitrary file copy as server process |
| CVE-2020-9273 | ProFTPD < 1.3.7a | Use-after-free — potential RCE |
| CVE-2022-22836 | CoreFTP Server | Directory traversal — write arbitrary files via PUT |

## Gotchas & Notes

- `hide_ids=YES` replaces real UIDs/GIDs with `ftp` in directory listings — you cannot determine file ownership. Without this, UID/GID values are visible and can be used to match users on other systems.
- `ls_recurse_enable=YES` lets you dump the entire directory tree with `ls -R` in a single command. Extremely useful when time is limited.
- Uploading files to an FTP server connected to a web server can lead to RCE if you can trigger execution of the uploaded file (e.g., via LFI or direct URL access).
- FTP log poisoning can lead to RCE. If logs are accessible from another vector (e.g., LFI), injecting PHP code into login attempts can achieve code execution.
- SSL certificate from `openssl s_client` reveals hostnames, email addresses, and organizational structure — treat this like any certificate transparency record.
- `wget -m --no-passive` mirrors the entire FTP tree. This is noisy — it downloads every accessible file. Use with caution on production servers.
- The banner (response code 220) often reveals the server software and version. Note this even if anonymous login fails.

## Related Pages

- [[attack/ftp]] — exploitation: anonymous login, Medusa brute force, bounce attack, CVE-2022-22836
- [[enumeration/_overview]]
- [[tools/attack/medusa]]
- [[tools/enumeration/nmap]]

## Sources

- raw/footprinting/host_based_enumeration_ftp.md
