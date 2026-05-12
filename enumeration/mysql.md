---
tags: [enumeration, enumeration/mysql]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# MySQL Enumeration

MySQL enumeration: banner grabbing, nmap scripts, authentication, information_schema queries, and user privilege checks.

## Overview

MySQL is an open-source relational database management system (Oracle-owned). MariaDB is a community fork. MySQL uses **TCP port 3306** by default.

MySQL is typically not exposed to the internet (it's meant for local application use), but misconfigurations frequently make it accessible. When found externally, it often has weak or default credentials. When found internally, it can expose application data, credentials stored in plaintext, and configuration details.

## Key Concepts / Techniques

### MySQL Architecture

- Runs as a service under a dedicated `mysql` OS user
- Data stored in `.sql` files or `InnoDB` format in `/var/lib/mysql/`
- Client connects on TCP 3306; responds to SQL queries
- MariaDB is functionally identical for enumeration purposes

### Critical System Databases

| Database | Purpose |
|----------|---------|
| `information_schema` | Metadata about all databases, tables, columns, users |
| `mysql` | User accounts, permissions, plugin config |
| `performance_schema` | Runtime performance data |
| `sys` | Human-readable performance views |

`information_schema` is the primary target for enumeration — it contains every database name, table name, column name, and user privilege on the instance.

### Dangerous Settings

| Setting | Description |
|---------|-------------|
| `user` | Service account (if `root`, dangerous) |
| `password` | Password stored in plaintext in config file |
| `admin_address` | Listening address (0.0.0.0 = exposed) |
| `debug` | Verbose error output |
| `sql_warnings` | Info strings on INSERT warnings |
| `secure_file_priv` | Controls file import/export — empty = unrestricted |

Config file location: `/etc/mysql/mysql.conf.d/mysqld.cnf` (often world-readable).

## Commands / Syntax

```bash
# Nmap scan with all MySQL scripts
sudo nmap 10.129.14.128 -sV -sC -p3306 --script mysql*

# Connect as root with empty password
mysql -u root -h 10.129.14.132

# Connect with password
mysql -u root -pP4SSw0rd -h 10.129.14.128

# Basic enumeration queries
MySQL> show databases;
MySQL> use mysql;
MySQL> show tables;
MySQL> show columns from user;

# Enumerate users and privileges
MySQL> select user, host, authentication_string from mysql.user;
MySQL> select * from information_schema.user_privileges;

# Check current user and permissions
MySQL> select user();
MySQL> select @@version;
MySQL> show grants;

# List all databases
MySQL> select schema_name from information_schema.schemata;

# List all tables in a database
MySQL> select table_name from information_schema.tables where table_schema='<dbname>';

# List columns in a table
MySQL> show columns from <table>;
MySQL> select column_name, data_type from information_schema.columns where table_name='<table>';

# Dump data
MySQL> select * from <table>;
MySQL> select * from <table> where <column> = "<value>";

# File read (if FILE privilege granted)
MySQL> select load_file('/etc/passwd');

# Host-based connection summary
MySQL> select host, unique_users from sys.host_summary;
```

## Flags & Options

### Nmap MySQL Scripts

| Script | Description |
|--------|-------------|
| `mysql-info` | Banner, capabilities, auth plugin |
| `mysql-empty-password` | Check for accounts with no password |
| `mysql-enum` | Enumerate valid usernames |
| `mysql-brute` | Credential brute-force |
| `mysql-databases` | List databases (requires credentials) |
| `mysql-users` | List users (requires credentials) |
| `mysql-variables` | List server variables |
| `mysql-dump-hashes` | Dump password hashes |

### MySQL Client Flags

| Flag | Description |
|------|-------------|
| `-u <user>` | Username |
| `-p<password>` | Password (no space between -p and password) |
| `-h <host>` | Remote host |
| `-P <port>` | Port (default 3306) |
| `-e "<query>"` | Execute query non-interactively |

## Version Detection & Exploit Research

MySQL exposes its version in the initial connection banner (pre-authentication) as part of the server greeting packet. `nmap -sV` and the `mysql-info` NSE script both retrieve it without credentials. The version determines applicable CVEs — MySQL and MariaDB have separate version tracks and separate CVE histories despite near-identical syntax.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap service scan | `nmap -sV -p3306 <IP>` | `MySQL 5.7.32` or similar from banner |
| Nmap mysql-info | `nmap -p3306 --script mysql-info <IP>` | Full banner + auth plugin + capabilities |
| SQL query (authenticated) | `SELECT @@version;` | Full version string |
| SQL query (authenticated) | `SELECT @@version_compile_os;` | OS the server was compiled for |
| SQL query (authenticated) | `SHOW VARIABLES LIKE '%version%';` | All version-related variables |

**MySQL vs MariaDB version strings:**
- `5.7.32-0ubuntu0.18.04.1` → MySQL 5.7 on Ubuntu
- `8.0.28-0ubuntu0.20.04.3` → MySQL 8.0
- `10.6.12-MariaDB` → MariaDB (separate CVE track from MySQL)

### Searching for Exploits

```bash
# Searchsploit
searchsploit mysql
searchsploit mysql 5.7
searchsploit mariadb

# Metasploit
msf6> search type:exploit name:mysql
msf6> search type:auxiliary name:mysql
```

### Notable CVEs

| CVE | Affected Versions | Impact |
|-----|------------------|--------|
| CVE-2012-2122 | MySQL 5.1.x, 5.5.x < 5.5.24 | Authentication bypass — repeated login attempts occasionally succeed due to memcmp flaw |
| CVE-2016-6662 | MySQL 5.5.x < 5.5.52, 5.6.x < 5.6.33, 5.7.x < 5.7.15 | Authenticated RCE via my.cnf injection |
| CVE-2016-6663 | MySQL, MariaDB (multiple) | Local privilege escalation via race condition in mysqld_safe |
| CVE-2021-27928 | MariaDB < 10.5.9 | RCE via `wsrep_provider` — requires authenticated access |
| CVE-2023-21980 | MySQL 8.0.x < 8.0.33 | Unauthenticated RCE via MySQL X Protocol |

## Gotchas & Notes

- **No space between `-p` and password**: `mysql -u root -pP4SSw0rd` is correct; `mysql -u root -p P4SSw0rd` prompts interactively.
- **Nmap can return false positives**: The `mysql-empty-password` script has known false-positive behavior. Always verify manually before reporting.
- **Config file leaks credentials**: `/etc/mysql/mysql.conf.d/mysqld.cnf` often contains the MySQL service user password in plaintext. If you have file read access (via another vulnerability), check this path.
- **`secure_file_priv` controls file operations**: If empty or set to a writable path, `LOAD DATA INFILE` and `INTO OUTFILE` can read/write arbitrary files.
- **WordPress and similar apps**: MySQL databases backing web apps almost always contain plaintext or MD5-hashed user credentials in the `users` or `wp_users` table.
- **MariaDB vs MySQL**: Authentication plugin differs (MariaDB uses `ed25519` by default in newer versions). The `caching_sha2_password` plugin in MySQL 8+ may require special client support.
- **SQL injection → MySQL enumeration**: If you find SQLi, the above queries work through the injection point — `information_schema` is your best friend for blind/union-based injection.

## Related Pages

- [[enumeration/_overview]]
- [[enumeration/mssql]]
- [[enumeration/oracle_tns]]

## Sources

- raw/footprinting/host_based_enumeration_mysql.md
