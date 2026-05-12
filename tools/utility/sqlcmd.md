---
tags: [tool]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# sqlcmd

Microsoft's official command-line client for SQL Server, used to run queries and scripts against MSSQL instances from both Linux and Windows.

## Overview

sqlcmd connects to SQL Server instances and executes T-SQL statements, batch scripts, and stored procedures. Pentesters use it to authenticate to discovered MSSQL services, enumerate databases, extract credentials, and execute OS commands via `xp_cmdshell` when enabled.

On Linux it requires the `mssql-tools18` package from Microsoft's repository. On Windows it ships with SQL Server, SSMS, or the standalone command-line utilities package. For Linux pentesting, `mssqlclient.py` from [[tools/utility/impacket]] is generally preferred — sqlcmd is most useful when a script expects it or when operating from a Windows pivot host.

Batch mode: statements are not executed until `GO` appears on its own line. This is the most common source of confusion for users coming from MySQL/psql.

## Installation

```bash
# Check if installed
sqlcmd -? 2>/dev/null | head -1 || echo "not installed"

# Install — add Microsoft repo and install mssql-tools18
curl https://packages.microsoft.com/keys/microsoft.asc \
    | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
curl https://packages.microsoft.com/config/debian/11/prod.list \
    | sudo tee /etc/apt/sources.list.d/mssql-release.list
sudo apt-get update
sudo ACCEPT_EULA=Y apt-get install -y msodbcsql18 mssql-tools18

# Add to PATH (add to ~/.bashrc for persistence)
echo 'export PATH="$PATH:/opt/mssql-tools18/bin"' >> ~/.bashrc
source ~/.bashrc

# Verify
sqlcmd -?
```

Note: `mssqlclient.py` from Impacket is preferred for Linux pentesting. sqlcmd is useful when scripts expect it, or when testing from a Windows pivot host.

## Usage

### Connect to a SQL Server instance

```bash
# SQL Server authentication
sqlcmd -S 10.129.201.248 -U sa -P 'password'

# Windows (domain) authentication — from a Windows host
sqlcmd -S 10.129.201.248 -E

# Named instance
sqlcmd -S 10.129.201.248\INSTANCE -U sa -P 'password'

# Non-standard port
sqlcmd -S 10.129.201.248,1433 -U sa -P 'password'
```

### Common enumeration queries

```sql
-- List all databases
SELECT name FROM sys.databases
GO

-- Use a specific database
USE master
GO

-- List tables in current database
SELECT table_name FROM information_schema.tables
GO

-- Show current user and permissions
SELECT SYSTEM_USER, USER_NAME()
GO

-- Check SQL Server version
SELECT @@version
GO

-- List SQL logins
SELECT name, type_desc FROM sys.server_principals WHERE type IN ('S','U','G')
GO
```

### Enable and use xp_cmdshell

```sql
-- Check if xp_cmdshell is enabled
SELECT value FROM sys.configurations WHERE name = 'xp_cmdshell'
GO

-- Enable xp_cmdshell (requires sysadmin)
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
GO

-- Execute OS command
EXEC xp_cmdshell 'whoami'
GO

-- Read a file
EXEC xp_cmdshell 'type C:\Windows\win.ini'
GO
```

### Run a query and exit (non-interactive)

```bash
# Single query, exit immediately
sqlcmd -S 10.129.201.248 -U sa -P 'password' -Q "SELECT name FROM sys.databases"

# Run a script file
sqlcmd -S 10.129.201.248 -U sa -P 'password' -i script.sql

# Output to file
sqlcmd -S 10.129.201.248 -U sa -P 'password' -Q "SELECT name FROM sys.databases" -o output.txt
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-S <server>` | Target server (IP, hostname, or IP,port) |
| `-U <user>` | SQL Server login username |
| `-P <pass>` | Password |
| `-E` | Use Windows (trusted/domain) authentication |
| `-Q "<query>"` | Run query and exit |
| `-q "<query>"` | Run query, stay in interactive mode |
| `-i <file>` | Read input from SQL script file |
| `-o <file>` | Write output to file |
| `-y 30` | Variable-length column display width |
| `-Y 30` | Fixed-length column display width |
| `-d <db>` | Default database to connect to |
| `-l <N>` | Login timeout in seconds |
| `-t <N>` | Query timeout in seconds |
| `-W` | Remove trailing spaces from output |
| `-h -1` | Remove column headers from output |

Use `-y 30 -Y 30` together to prevent wide columns from truncating output.

## Gotchas & Notes

- **GO is required**: T-SQL statements are not executed until `GO` appears on its own line. Forgetting `GO` is the most common reason nothing happens.
- **xp_cmdshell is disabled by default** in SQL Server 2005+. Enabling it requires `sysadmin` and leaves a trace in the SQL Server error log.
- **ACCEPT_EULA** must be set during install on Linux. The package will not install without it.
- **mssql-tools18 vs mssql-tools**: The newer `mssql-tools18` package uses ODBC Driver 18 which enforces TLS encryption by default. Use `-C` or set `TrustServerCertificate=yes` in the connection string if the server has a self-signed certificate.
- **PATH**: The tools install to `/opt/mssql-tools18/bin/`, not a standard system path. Always confirm `sqlcmd` is on PATH after install.
- **Linked servers**: If `xp_cmdshell` is disabled, enumerate linked servers (`SELECT * FROM sys.servers`) — a linked server may have it enabled.
- **Output column width**: Without `-y 30 -Y 30`, long values (e.g. file contents, query results) are truncated silently.

## Related Pages

- [[attack/sql_databases]]
- [[tools/utility/impacket]]
- [[labs/htb/attacking_common_services/mssql_hash_theft_and_db_enumeration]]

## Sources

- raw/attacking_common_services/sql_attacking_sql_databases.md
