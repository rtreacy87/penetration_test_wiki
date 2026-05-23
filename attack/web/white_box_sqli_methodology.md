---
tags: [attack/web, attack, concept]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 3
---

# White-Box SQLi Methodology

How to approach SQL injection discovery when you have access to the application binary or source code. Covers decompiling Java JARs, grepping for vulnerable patterns, enabling PostgreSQL query logging, and attaching a remote debugger.

## Overview

White-box testing compresses the recon phase dramatically. Rather than fuzzing every parameter blindly, you:
1. Extract/read the source code
2. Grep for SQL string concatenation patterns
3. Trace user-controlled input to vulnerable sinks
4. Confirm and exploit

The application in these examples is **BlueBird** — a Java Spring Boot app backed by PostgreSQL, distributed as a compiled JAR.

---

## Step 1 — Decompile the JAR

Java bytecode in `.jar` files can be decompiled back to readable `.java` source. Two tools:

### Fernflower (CLI — preferred)

Maintained by JetBrains. Produces clean source output. CLI-only — suitable for remote/SSH work.

```bash
# Clone and build (requires JDK 17)
git clone https://github.com/fesh0r/fernflower.git
cd fernflower

# If build fails with "invalid source release: 17", install JDK 17 first:
sudo apt install openjdk-17-jdk -y
sudo update-java-alternatives --list   # find the path
sudo update-java-alternatives --set /usr/lib/jvm/java-1.17.0-openjdk-amd64

./gradlew build
# Output: build/libs/fernflower.jar

# Decompile target JAR
mkdir out
java -jar build/libs/fernflower.jar /opt/bluebird/BlueBird-0.0.1-SNAPSHOT.jar out/

# Extract .java files from decompiled JAR
cd out/
jar -xf BlueBird-0.0.1-SNAPSHOT.jar
# Source files now in: BOOT-INF/classes/
```

> **Note (2025):** Fernflower now targets JDK 21 on the latest commit. If building on JDK 17, run `git checkout c03cae7` before `./gradlew build`.

### JD-GUI (GUI — legacy)

A graphical JAR decompiler. Last release was 2019; treat as legacy.

```bash
# Download from: https://github.com/java-decompiler/jd-gui/releases
java -jar jd-gui-1.6.6.jar BlueBird-0.0.1-SNAPSHOT.jar
# GUI opens; File → Save All Sources → extracts a ZIP of .java files
```

**Prefer Fernflower** — JD-GUI is discontinued and its GUI is unnecessary for source extraction.

---

## Step 2 — Grep for Vulnerable SQL Patterns

With source files in hand, use `grep` to locate SQL string concatenation (the precondition for injection).

### Regex patterns

| Pattern | Finds |
|---------|-------|
| `SELECT\|UPDATE\|DELETE\|INSERT\|CREATE\|ALTER\|DROP` | All SQL statements |
| `(WHERE\|VALUES).*?' ` | WHERE/VALUES followed by a single quote → string concat |
| `(WHERE\|VALUES).*?" \+` | WHERE/VALUES with Java string concat |
| `.*sql.*"` | Lines with a variable named `sql` and a string |
| `jdbcTemplate` | All JdbcTemplate calls (Spring Boot ORM) |

```bash
# Find all string-concatenated SQL queries
grep -irnE '(WHERE|VALUES).*" +' --include='*.java' .

# Example output:
# ./controller/AuthController.java:134: String sql = "SELECT * FROM users WHERE email = '" + email + "'";

# Find all JdbcTemplate usages
grep -nrE 'jdbcTemplate' --include='*.java' .

# Find all SQL variable assignments
grep -nrE '.*sql.*"' --include='*.java' .
```

### What to look for

For each SQL concatenation found, answer:
1. **Is any concatenated value user-controlled?** Trace the variable back to `@RequestParam`, path variables, or database values loaded from user-controlled data.
2. **Does any validation regex have a logic bug?** See [[attack/web/advanced_sqli_character_bypasses]] — `Matcher.matches()` vs `Matcher.find()` is a common Java-specific trap.
3. **Is this a second-order sink?** The value being concatenated may have come from the database, not directly from the HTTP request — see [[attack/web/second_order_sqli]].

---

## Step 3 — Enable PostgreSQL Query Logging

Logging all executed queries lets you see exactly what SQL reaches the database — invaluable for payload debugging.

```bash
# Find postgresql.conf
find / -type f -name postgresql.conf 2>/dev/null
# Typically: /etc/postgresql/<version>/main/postgresql.conf

# Edit postgresql.conf — make these changes:
sudo nano /etc/postgresql/13/main/postgresql.conf
```

Changes to make:

```ini
logging_collector = on          # was: #logging_collector = off
log_statement = 'all'           # was: #log_statement = 'none'
log_directory = 'pg_log'        # uncomment — sets log folder relative to data dir
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  # uncomment
```

```bash
# Apply changes
sudo systemctl restart postgresql

# Watch live query log
sudo watch -n1 tail /var/lib/postgresql/13/main/pg_log/postgresql-*.log

# Or stream continuously
sudo tail -f /var/lib/postgresql/13/main/pg_log/postgresql-*.log
```

**What to look for:**
- Confirm your payload reaches the database unmodified
- See the full assembled query to understand column counts, types, and structure
- Spot when filters strip or transform your input before it hits the DB

---

## Step 4 — Input Tracing

Before exploiting, confirm the injection path end-to-end:

1. **Find the sink** — the SQL query with concatenation (from grep)
2. **Trace backwards** — follow the variable up through function arguments
3. **Find the source** — which HTTP parameter, path variable, or DB field feeds it
4. **Check validation** — what regex/filter sits between source and sink
5. **Test bypass** — does the filter have a logic flaw? (see [[attack/web/advanced_sqli_character_bypasses]])

For second-order: look for `UPDATE` statements that write user input to the DB, then check if any `SELECT` later concatenates those DB values unsafely.

```bash
# Find UPDATE queries that include user-supplied email, name, etc.
grep -irnE 'UPDATE.*email' --include='*.java' .

# Trace a specific variable back to its source
grep -nrE 'getEmail\(\)' --include='*.java' .
```

---

## Step 5 — Remote Live Debugging (Optional)

Attach a debugger to see variable values in real-time as the application processes your request.

### CLI: JDB (Java Debugger)

```bash
# On the target server — start the app in debug mode
java -Xdebug -Xrunjdwp:transport=dt_socket,address=8000,server=y,suspend=y \
  -jar /opt/bluebird/BlueBird-0.0.1-SNAPSHOT.jar

# Forward the debug port via SSH
ssh -L 8000:127.0.0.1:8000 student@<target-ip>

# Attach JDB from local machine
jdb -attach 127.0.0.1:8000

# Set a breakpoint at a specific line
stop at com.bmdyy.bluebird.controller.IndexController:71

# Continue execution
run
cont

# When breakpoint hits, inspect variables
print sql
print u
```

### GUI: VSCode or Eclipse (Remote Attach)

Both IDEs support remote Java debugging over JDWP. Open the decompiled source folder, add the dependency JARs from `BOOT-INF/lib/` as referenced libraries, create a `launch.json` with `"request": "attach"` and `"port": 8000`, then press F5 after SSH-forwarding the port. Set breakpoints by clicking the gutter.

**Prefer JDB** for headless/SSH environments — VSCode/Eclipse require a local GUI.

---

## Typical Vulnerability Locations in Spring Boot Apps

| Controller / Endpoint | Pattern to look for |
|----------------------|---------------------|
| Search/filter endpoints | `LIKE '%' + input + '%'` |
| Password reset (`/forgot`) | `WHERE email = '` + email |
| Profile pages (`/profile/{id}`) | Concatenation of DB-sourced values (second-order) |
| Admin panels | UPDATE/INSERT with user-controlled fields |

---

## Related Pages

- [[attack/web/advanced_sqli_character_bypasses]] — exploiting injection once the sink is identified
- [[attack/web/error_based_sqli]] — verbose error extraction via CAST
- [[attack/web/second_order_sqli]] — exploiting stored/second-order injection
- [[attack/web/postgresql_file_read_write]] — post-injection file access
- [[attack/web/postgresql_rce]] — post-injection command execution
- [[attack/web/sqli_prevention]] — parameterized queries and least-privilege fixes
- [[tools/utility/psql]] — PostgreSQL CLI for payload testing

## Sources

- raw/advanced_sql_injections/identifying_vulnerabilities_decompiling_java_archives.md
- raw/advanced_sql_injections/identifying_vulnerabilities_searching_for_strings.md
- raw/advanced_sql_injections/identifying_vulnerabilities_hunting_for_sql_errors.md
- raw/advanced_sql_injections/identifying_vulnerabilities_live_debugging_java_applications.md
