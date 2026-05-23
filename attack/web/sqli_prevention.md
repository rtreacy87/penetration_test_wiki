---
tags: [attack/web, concept, definition]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# SQL Injection Prevention

How to eliminate SQL injection vulnerabilities in code and reduce the blast radius when injection does occur. Covers parameterized queries, least-privilege database users, and permission hardening for PostgreSQL-specific attack vectors.

## Primary Fix: Parameterized Queries

The only reliable defence against SQL injection is to **never concatenate user input into SQL strings**. Use parameterized queries (also called prepared statements) instead — the database receives query structure and data as separate inputs and cannot confuse one for the other.

### Java + Spring JdbcTemplate

```java
// VULNERABLE — string concatenation
String sql = "SELECT * FROM users WHERE username LIKE '%" + u + "%'";
List<User> users = jdbcTemplate.query(sql, new BeanPropertyRowMapper(User.class));

// FIXED — parameterized
String sql = "SELECT * FROM users WHERE username LIKE CONCAT('%', ?, '%')";
List<User> users = jdbcTemplate.query(sql, new Object[]{u}, new BeanPropertyRowMapper(User.class));
```

The `?` placeholder prevents the value of `u` from ever being interpreted as SQL, regardless of what characters it contains.

### Other Languages

```python
# Python + psycopg2
cur.execute("SELECT * FROM users WHERE email = %s", (email,))

# Python + SQLAlchemy
result = session.execute(text("SELECT * FROM users WHERE email = :email"), {"email": email})
```

```javascript
// Node.js + pg
client.query('SELECT * FROM users WHERE email = $1', [email])
```

```php
// PHP + PDO
$stmt = $pdo->prepare("SELECT * FROM users WHERE email = ?");
$stmt->execute([$email]);
```

### ORM Equivalents

ORMs (Hibernate, SQLAlchemy, ActiveRecord) use parameterized queries by default for their standard query API. Injection typically re-enters via raw query escape hatches (`Session.execute(raw_sql)`, `Model.where(raw_string)`). Audit every raw query call in an ORM-using codebase.

---

## Secondary Fix: Principle of Least Privilege

Even with parameterized queries, restrict what the database user can do. If an attacker finds an unpatched injection, limited privileges reduce the blast radius.

### Rule: Grant only what the application needs

```sql
-- Create a restricted application user
CREATE USER appuser WITH PASSWORD 'strong_password';

-- Grant only required table permissions
GRANT SELECT, INSERT, UPDATE ON users TO appuser;
GRANT SELECT, INSERT ON posts TO appuser;

-- Do NOT grant superuser, CREATEROLE, or CREATEDB
```

### PostgreSQL: Restrict file operations

```sql
-- Revoke roles that enable file read/write/execution
REVOKE pg_read_server_files FROM appuser;
REVOKE pg_write_server_files FROM appuser;
REVOKE pg_execute_server_program FROM appuser;

-- These are not granted by default but verify:
SELECT r.rolname, ARRAY(
  SELECT b.rolname FROM pg_catalog.pg_auth_members m
  JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
  WHERE m.member = r.oid
) FROM pg_catalog.pg_roles r WHERE r.rolname = 'appuser';
```

**Implication:** without `pg_execute_server_program` or superuser, COPY FROM PROGRAM (see [[attack/web/postgresql_rce]]) is blocked. Without `pg_read_server_files` / `pg_write_server_files`, COPY FROM/TO file paths are blocked.

### PostgreSQL: Restrict large objects

```sql
-- Revoke default large object access (PostgreSQL 9.0+)
REVOKE ALL ON LARGE OBJECTS FROM PUBLIC;
REVOKE ALL ON LARGE OBJECTS FROM appuser;

-- Grant only if the application genuinely needs large objects
GRANT SELECT ON LARGE OBJECTS TO appuser;   -- read
GRANT UPDATE ON LARGE OBJECTS TO appuser;   -- write
```

**Implication:** without large object write access, the extension upload chain (see [[attack/web/postgresql_rce]] Method 2) cannot proceed.

### PostgreSQL: Restrict extension creation

```sql
-- Extension creation requires CREATE privilege on the schema
REVOKE CREATE ON SCHEMA public FROM appuser;
```

**Implication:** without CREATE on the schema, `CREATE FUNCTION ... LANGUAGE C` fails, blocking the custom extension RCE path entirely.

---

## Defence in Depth — Stack

| Layer | Control | Blocks |
|-------|---------|--------|
| Code | Parameterized queries | All SQLi at the injection point |
| DB user | No superuser | COPY FROM PROGRAM, COPY file R/W, extensions |
| DB user | No `pg_read_server_files` | COPY file reads |
| DB user | No `pg_write_server_files` | COPY file writes / webshell placement |
| DB user | No `pg_execute_server_program` | COPY FROM PROGRAM RCE |
| Schema | No CREATE on public | Custom C extension RCE |
| Large objects | No UPDATE privilege | Extension upload via large objects |
| Network | PostgreSQL not exposed externally | Direct psql connections from attackers |

---

## Testing Your Fixes

After patching, re-run the original PoC payloads to confirm they no longer work:

```bash
# Test character bypass payload (should return empty or normal results, not injected data)
curl 'http://target:8080/find-user?u=%27/**/and/**/1=1--'

# Test UNION payload via second-order (should store literally, not inject)
curl -X POST 'http://target:8080/profile/edit' \
  --data "name=test&description=test&email=' UNION SELECT '1','2','3','4',5--"

# Verify the stored value did not inject on trigger
curl 'http://target:8080/profile/42'
```

---

## Gotchas & Notes

- **Input validation is not a substitute for parameterized queries.** Regex filters can be bypassed (see [[attack/web/advanced_sqli_character_bypasses]]). Parameterized queries cannot.
- **ORMs are not automatically safe.** Any raw query method in an ORM is vulnerable to the same injection as manual SQL.
- **Second-order injection bypasses storage-side input validation entirely.** The parameterized UPDATE stores the payload literally; the bug is in the unsafe SELECT that runs later. You must fix the SELECT, not the UPDATE.
- **Error messages leak schema information.** Even with all the above fixes, verbose error pages help attackers understand the database structure. Disable stack traces in production responses.

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — how to audit code for all three injection patterns
- [[attack/web/advanced_sqli_character_bypasses]] — why input validation alone fails
- [[attack/web/second_order_sqli]] — the attack this page's second-order fix addresses
- [[attack/web/postgresql_file_read_write]] — attack vector blocked by file-role revocation
- [[attack/web/postgresql_rce]] — attack vector blocked by COPY PROGRAM + extension controls
- [[definitions/security_terminology]] — SQLi definition

## Sources

- raw/advanced_sql_injections/preventing_sql_injection_vulnerabilities.md
