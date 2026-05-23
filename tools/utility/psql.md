---
tags: [tool, enumeration, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# psql

The official PostgreSQL interactive terminal. Ships with every PostgreSQL server installation and is the primary CLI tool for interacting with Postgres databases — both for administration and for manual injection testing.

> **pgAdmin4** is a GUI alternative to psql. It is useful for visual schema browsing but is cumbersome during injection testing. Prefer psql for all pentest work; use pgAdmin4 only if you need its query plan visualiser or ER diagram features.

## Installation

```bash
# Check if installed
psql --version

# Install (Kali / Parrot) — client only, no server needed
sudo apt install postgresql-client-15 -y

# If version 15 is unavailable, version 13 works for all lab steps
sudo apt install postgresql-client-13 -y

# Verify
psql --version
```

---

## Connecting

```bash
# Basic connection
psql -h 127.0.0.1 -U acdbuser acmecorp
# Password prompt appears

# With port (default 5432)
psql -h 127.0.0.1 -p 5432 -U acdbuser acmecorp

# Connection string form
psql "postgresql://acdbuser:AcmeCorp2023!@127.0.0.1:5432/acmecorp"

# Non-interactive single query (-c flag)
psql -h 127.0.0.1 -U acdbuser acmecorp -c "SELECT version();"

# Run a SQL file
psql -h 127.0.0.1 -U acdbuser acmecorp -f queries.sql
```

---

## Essential Meta-Commands

Meta-commands start with `\` and are processed by psql, not sent to the server.

| Command | Effect |
|---------|--------|
| `\l` | List all databases |
| `\l+` | List databases with size/owner detail |
| `\c <db>` | Switch to database |
| `\dt` | List tables in current database |
| `\dt+` | Tables with size and description |
| `\d <table>` | Describe a table (columns, types, constraints) |
| `\du` | List roles/users |
| `\dn` | List schemas |
| `\df` | List functions |
| `\x` | Toggle expanded display (useful for wide rows) |
| `\timing` | Show query execution time |
| `\copy` | Client-side COPY (works as any user; server-side COPY requires superuser) |
| `\q` | Quit |

---

## Common Pentest Queries

```sql
-- Current user and privileges
SELECT current_user, session_user;
SELECT current_setting('is_superuser');

-- Check if user has a specific role
SELECT r.rolname, ARRAY(
  SELECT b.rolname FROM pg_catalog.pg_auth_members m
  JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
  WHERE m.member = r.oid
) AS memberof
FROM pg_catalog.pg_roles r WHERE r.rolname = 'targetuser';

-- List all databases
SELECT datname FROM pg_database;

-- List all tables in current DB
SELECT table_name FROM information_schema.tables
WHERE table_schema = 'public';

-- Dump column names for a table
SELECT column_name, data_type FROM information_schema.columns
WHERE table_name = 'users';

-- PostgreSQL version
SELECT version();

-- Database server file paths (superuser only)
SHOW data_directory;
SHOW config_file;

-- List all extensions
SELECT * FROM pg_extension;
```

---

## File Operations (Superuser / Privileged Role Required)

```sql
-- Read a file via COPY
CREATE TABLE tmp(t TEXT);
COPY tmp FROM '/etc/passwd' DELIMITER E'\x07';
SELECT * FROM tmp;
DROP TABLE tmp;

-- Write a file via COPY
CREATE TABLE tmp(t TEXT);
INSERT INTO tmp VALUES('payload');
COPY tmp TO '/tmp/output.txt';
DROP TABLE tmp;

-- Import file as large object
SELECT lo_import('/etc/passwd');        -- returns OID
SELECT lo_get(<OID>);                   -- hex output; decode with xxd -r -p
SELECT lo_unlink(<OID>);               -- cleanup
```

---

## pgAdmin4 — GUI Alternative

pgAdmin4 is a web-based GUI for PostgreSQL installed locally. Recommended only when visual schema exploration outweighs the friction of a browser:

```bash
# Install pgAdmin4 (Kali/Parrot — use 'bullseye' as release name)
curl -fsS https://www.pgadmin.org/static/packages_pgadmin_org.pub \
  | sudo gpg --dearmor -o /usr/share/keyrings/packages-pgadmin-org.gpg
sudo sh -c 'echo "deb [signed-by=/usr/share/keyrings/packages-pgadmin-org.gpg] \
  https://ftp.postgresql.org/pub/pgadmin/pgadmin4/apt/bullseye pgadmin4 main" \
  > /etc/apt/sources.list.d/pgadmin4.list && apt update'
sudo apt install pgadmin4 -y
```

Access via browser at `http://127.0.0.1/pgadmin4`. Use `Dashboard → Add New Server` to connect.

**Prefer psql for injection testing** — psql allows raw query execution, fast iteration, and scripting; pgAdmin4 is slower for payload development.

---

## Gotchas & Notes

- Queries must end with `;` in psql. Without it, psql waits for continuation and shows `->`.
- `\copy` (client-side) works as any user; server-side `COPY` requires `pg_read_server_files`, `pg_write_server_files`, or superuser.
- `psql` expands `~` and tab-completes table/column names — useful for live testing.
- When connecting to a target via SSH tunnel: `ssh -L 5432:127.0.0.1:5432 user@target` then `psql -h 127.0.0.1 -U ...`.

## Related Pages

- [[attack/web/postgresql_file_read_write]] — COPY and Large Objects file operations
- [[attack/web/postgresql_rce]] — COPY FROM PROGRAM and custom extensions
- [[attack/web/advanced_sqli_character_bypasses]] — payload encoding for PostgreSQL
- [[attack/web/error_based_sqli]] — CAST-to-INT error leakage
- [[attack/web/second_order_sqli]] — stored injection via profile fields
- [[attack/sql_databases]] — MSSQL equivalent techniques

## Sources

- raw/advanced_sql_injections/introduction_to_postgresql.md
