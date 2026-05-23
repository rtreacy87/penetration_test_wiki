---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Hunting for SQL Errors

Target access: SSH to the target with `student:academy.hackthebox.com`.

This lab is entirely CLI-based. There are no GUI tools involved — the task is reading PostgreSQL log files on the target system via SSH.

## Tool Installation

No special tools needed beyond a standard SSH client. See [[tools/utility/ssh]] for connection options including the `vssh` alias for fingerprint-free VM connections.

---

## Question 1 — Value of `application_name` when BlueBird starts up

**Goal:** Find the value PostgreSQL logs for `application_name` when the BlueBird application connects.

**Why look at log files at all:** When you have SSH access to a target running a web application backed by PostgreSQL, log files are a high-value reconnaissance target. They tell you things you can't easily get from the application itself: the actual SQL queries the application generates (including parameterized query templates), which database username the app connects as, what driver and version is in use, and — critically — any queries that failed with errors. Error messages in logs often expose table names, column names, and type mismatches that would take many manual probes to discover through the application. This is often faster and stealthier than black-box fuzzing.

**Step 1 — SSH to the target**

```bash
ssh student@<TARGET_IP>
# Password: academy.hackthebox.com
# or using vssh for fingerprint-free connection:
vssh student@<TARGET_IP>
```

Once you have a shell, resist the urge to immediately start probing the web app. With server access, you can gather far more information locally before touching the application layer.

**Step 2 — Locate the PostgreSQL log directory**

PostgreSQL logging has already been enabled on this VM. Before searching, find out where the logs are:

```bash
ls /opt/bluebird/
```

The directory contains both the application JAR and a `pg_log/` subdirectory. This is a non-default log location — production deployments often put PostgreSQL logs under `/var/log/postgresql/` or the data directory. Always check application-specific directories too.

**Why is the log here and not under /var/log:** The application was likely configured to log to its own directory for centralized management. In a real engagement, check both locations, and also check the PostgreSQL config for the actual log_directory setting:

```bash
# If you can run psql as the postgres user
sudo -u postgres psql -c "SHOW log_directory;"
```

**Step 3 — Search for application_name**

The specific value `application_name` appears in connection logs when a client sets it as part of its connection handshake. JDBC drivers do this automatically. You don't need to read every log file — `grep -r` searches all of them at once:

```bash
grep -i "application_name" -r /opt/bluebird/pg_log/ -m 1 | head -n 1
```

`-i` for case-insensitive (log formats vary), `-r` for recursive, `-m 1` to stop after the first match and avoid reading thousands of duplicate entries.

You may see a `binary file matches` message for one of the log files — this is a binary-format PostgreSQL log. Skip it and read the text log file. The text log will show something like:

```
2023-03-22 14:45:37.995 EDT [1072] bbuser@bluebird LOG:  execute <unnamed>: SET application_name = 'PostgreSQL JDBC Driver'
```

**What this tells you beyond the answer:**
- `bbuser@bluebird` — the database username and database name the application uses. If you later find a way to authenticate directly to PostgreSQL (via SQLi that reads credentials, or via exposed port), this is the account to target.
- `PostgreSQL JDBC Driver` — the exact JDBC driver string. This can be cross-referenced against CVE databases if you want to pursue driver-level vulnerabilities.
- The timestamp shows when BlueBird last restarted — useful if you're trying to understand when log rotation happened or how stale the data is.

The value of `application_name` is `PostgreSQL JDBC Driver`.

**What to look for beyond this question:** While you have grep open on the log directory, it's worth running a few more searches before moving on:

```bash
# Look for SQL errors (potential injection points or failed auth)
grep -i "ERROR\|FATAL" -r /opt/bluebird/pg_log/ | tail -20

# Look for actual query patterns
grep -i "execute\|statement" -r /opt/bluebird/pg_log/ | grep -v "application_name" | head -20
```

These aren't required for the question but give you a map of the application's query patterns — which directly helps when you're hunting for injectable queries in the source code.

---

## Why This Matters

When attacking a white-box application, PostgreSQL log files can reveal:
- Which queries the application executes (including parameterized query templates)
- Connection parameters and driver versions
- Error messages that expose table or column names
- The database username the application runs as

In a black-box scenario without server access, you'd have to manually probe every input field and trigger errors to discover the same information. Log access collapses that work to a single grep.

---

## Lessons Learned

- Server access changes the attack model entirely — logs, config files, and the filesystem are all higher-value targets than HTTP fuzzing
- Binary log files require `strings` or a PostgreSQL log parser (like `pgbadger`) to read; text log files are directly greppable
- `application_name` in PostgreSQL logs identifies the connecting client — useful for profiling the application stack and finding driver CVEs
- The database username visible in logs (`bbuser`) is separate from the OS user running PostgreSQL — don't confuse them when planning subsequent privilege escalation

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — using log files as part of source code review
- [[attack/web/error_based_sqli]] — deliberately triggering errors to extract data
- [[tools/utility/ssh]] — SSH reference including vssh alias
