---
tags: [tool, enumeration, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# Fernflower

Java decompiler maintained by JetBrains and bundled inside IntelliJ IDEA. Converts compiled `.class` bytecode back into readable Java source code, making it the primary tool for white-box source code review on Java applications when the original source is unavailable.

## Overview

When you encounter a compiled Java application — a `.jar`, `.war`, or `.ear` file — during a white-box or grey-box engagement, Fernflower converts it back to human-readable Java. This lets you:

- Read every HTTP endpoint the application exposes (including undocumented ones)
- Find SQL queries built by string concatenation (injection sinks)
- Find hardcoded secrets: JWT keys, API tokens, database passwords
- Understand authentication and session handling logic
- Trace how user input flows through the application (source → sink analysis)

Fernflower produces source-accurate output — method signatures, class hierarchies, annotations, and loop structures are reconstructed faithfully rather than approximated. This makes it easier to grep for patterns and trace data flow than lower-fidelity disassemblers.

---

## When to reach for Fernflower

Fernflower is the right tool when all of the following are true:

- You have a compiled Java artifact (`.jar`, `.war`, `.ear`)
- The engagement is white-box or grey-box (you're permitted to reverse the artifact)
- You want to do systematic source code review, not just run automated scanners

**Indicators during an engagement that a Java JAR is present:**

```bash
# Service fingerprinting reveals a Java application server
nmap -sV -p 8080,8443,9090 <TARGET>
# Response: "Apache Tomcat", "Spring Boot", "Jetty", "JBoss", "WildFly"

# SCP/download the application directory from a compromised server
ls /opt/           # production apps often in /opt/<appname>/
ls /var/lib/       # data directories
ls /home/*/        # developer home directories

# The file command identifies JARs
file *.jar
# JAR archive (ZIP format): BlueBird-0.0.1-SNAPSHOT.jar
```

**Choosing Fernflower vs alternatives:**

| Tool | Best for | Notes |
|------|----------|-------|
| **Fernflower** | Full white-box review, accurate source reconstruction | Requires Java 21 JDK to build; output is closest to original source |
| **jadx** | Quick analysis, no build step required | `sudo apt install jadx -y`; slightly less accurate but faster to set up |
| **strings + grep** | Finding plaintext secrets without decompilation | Works on `.properties` and `.yml` files embedded in the JAR without decompilation |

Use Fernflower when you need accurate source structure for tracing data flow. Use jadx when you're on a time constraint or just searching for a specific pattern.

---

## Installation

### Verified working procedure on Kali rolling (Java 25 default)

Kali rolling ships with Java 25 EA as default and only JRE packages — no `javac` compiler. The Fernflower Gradle build requires a full JDK. `msopenjdk-21` is the verified working package: stable build, includes `javac`, and Fernflower's `build.gradle.kts` targets `sourceCompatibility = "21"`.

```bash
# Step 1 — Install Microsoft's Java 21 JDK
# (openjdk-21-jdk in Kali's repos is an early-access build that triggers a
#  Gradle 9 toolchain detection bug; msopenjdk-21 is the stable release)
sudo apt install msopenjdk-21 -y
# Installs to: /usr/lib/jvm/msopenjdk-21-amd64/
# Registers javac at /usr/bin/javac via update-alternatives automatically

# Step 2 — Clone the Fernflower mirror
git clone https://github.com/fesh0r/fernflower.git
cd fernflower/

# Step 3 — Build
# JAVA_HOME scopes the build to msopenjdk-21 without changing your system default
# --no-configuration-cache bypasses a Gradle 9 serialization bug with EA Java builds
JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 ./gradlew build -x test --no-configuration-cache
```

Expected output:
```
> Task :compileJava
> Task :classes
> Task :jar
> Task :build
BUILD SUCCESSFUL in 9s
```

```bash
# Step 4 — Verify
ls ~/fernflower/build/libs/fernflower.jar
```

### Troubleshooting the build

**Error: `Unable to locate package openjdk-17-jdk`**
`openjdk-17-jdk` was dropped from Kali's rolling repos. Use `msopenjdk-21` instead (covers Fernflower's Java 21 requirement since the September 2025 update).

**Error: `Toolchain installation '...' does not provide the required capabilities: [JAVA_COMPILER]`**

Two distinct causes with the same error message:

| Cause | Check | Fix |
|-------|-------|-----|
| Only JRE installed, no JDK | `which javac` returns nothing | `sudo apt install msopenjdk-21 -y` |
| JDK present but Gradle 9 can't profile it (EA build) | `javac --version` works but build fails anyway | Add `--no-configuration-cache` AND set `JAVA_HOME` to the stable msopenjdk-21 path |

**Alternative if Fernflower still won't build:**
```bash
sudo apt install jadx -y
# jadx is a capable substitute that requires no build step
```

---

## Basic Usage

### Decompile a JAR

Always set `JAVA_HOME` explicitly so Java 25 isn't picked up as the runtime:

```bash
mkdir <output_dir>/
JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 java \
    -jar ~/fernflower/build/libs/fernflower.jar \
    <source.jar> \
    <output_dir>/
```

Fernflower writes a new JAR to `<output_dir>/` containing the decompiled `.java` files. Extract it:

```bash
cd <output_dir>/
jar -xf <source.jar>
```

**Spring Boot fat JARs** unpack to:
```
BOOT-INF/
  classes/          ← application source (controllers, models, config)
  lib/              ← bundled dependencies (third-party JARs)
META-INF/
  MANIFEST.MF
org/springframework/ ← embedded Spring Boot launcher (ignore this)
```

Everything you care about is in `BOOT-INF/classes/`.

### Flags

| Flag | Description |
|------|-------------|
| (positional 1) | Source JAR/directory to decompile |
| (positional 2) | Output directory for the decompiled JAR |
| `-hes=0` | Don't hide empty super calls |
| `-hdc=0` | Don't hide default constructor |
| `-dgs=1` | Decompile generic signatures |
| `-rsy=1` | Synchronize method decompilation |
| `-log=WARN` | Suppress INFO-level decompilation messages (useful for large JARs) |

For source code review, defaults are fine — no extra flags needed.

---

## Systematic Enumeration of a Decompiled JAR

Work through this sequence after extracting the source. The goal is to build a complete picture of the application before touching it over the network.

### 1 — Orient yourself: map the application structure

```bash
# See what packages exist (reveals the application's domain model)
find BOOT-INF/classes -name "*.java" | head -40

# List all controllers (HTTP endpoint handlers in Spring Boot)
find BOOT-INF/classes -name "*Controller.java"

# List all models (database entities — tells you the table/column names)
find BOOT-INF/classes -name "*.java" -path "*/model/*"

# Find the main application class (entry point, reveals package structure)
find BOOT-INF/classes -name "*Application.java"

# Read application config — often contains DB credentials, JWT secrets, API keys
cat BOOT-INF/classes/application.properties 2>/dev/null
cat BOOT-INF/classes/application.yml 2>/dev/null
cat BOOT-INF/classes/application-prod.properties 2>/dev/null
```

The controller list tells you every HTTP endpoint. Read each controller file — they are typically small and fast to scan.

### 2 — Harvest hardcoded secrets

```bash
# Broad secret search across all files
grep -rn "secret\|password\|passwd\|token\|apikey\|api_key\|jwtSecret\|privateKey" \
    BOOT-INF/classes/ --include="*.java" --include="*.properties" --include="*.yml" -i

# JWT-specific (common in Spring Boot)
grep -rn "jwtSecret\|jwt.secret\|JWT_SECRET\|HS256\|HS512" BOOT-INF/classes/ -i

# Database connection strings
grep -rn "jdbc:\|datasource\|spring.datasource" BOOT-INF/classes/ -i

# Hardcoded credentials in test/seed data
grep -rn "password.*=.*['\"]" BOOT-INF/classes/ --include="*.java"
```

### 3 — Find SQL injection sinks

The goal is to find every place where a SQL query is built using string concatenation rather than parameterized queries.

```bash
# Find all SQL keywords — shows every query in the application
grep -rn "SELECT\|INSERT\|UPDATE\|DELETE\|WHERE\|JOIN" \
    BOOT-INF/classes/ --include="*.java" -l

# Find string concatenation in SQL context (the dangerous pattern)
# Look for: "SELECT..." + variable or "WHERE " + variable
grep -rn '".*SELECT.*"\|".*INSERT.*"\|".*UPDATE.*"\|".*WHERE.*"' \
    BOOT-INF/classes/ --include="*.java"

# Find JdbcTemplate usage (Spring's SQL execution method)
grep -rn "jdbcTemplate\|JdbcTemplate\|execute\|query\|queryFor" \
    BOOT-INF/classes/ --include="*.java"

# Find PreparedStatement (safe) vs Statement (potentially unsafe)
grep -rn "PreparedStatement\|createStatement\|\.execute(" \
    BOOT-INF/classes/ --include="*.java"
```

**What to look for in the output:** Any query where the string is built by concatenating a variable rather than using `?` placeholders. Example of the dangerous pattern:

```java
// UNSAFE — injectable
String sql = "SELECT * FROM users WHERE email = '" + email + "'";
jdbcTemplate.execute(sql);

// SAFE — parameterized
String sql = "SELECT * FROM users WHERE email = ?";
jdbcTemplate.queryForObject(sql, new Object[]{email}, User.class);
```

### 4 — Find authentication and authorization logic

```bash
# Find login/authentication endpoints
grep -rn "login\|authenticate\|signin\|@PostMapping" \
    BOOT-INF/classes/ --include="*.java" -l

# Find JWT generation and validation
grep -rn "generateToken\|validateToken\|parseJwt\|Jwts\." \
    BOOT-INF/classes/ --include="*.java"

# Find authorization checks — look for places these are MISSING
grep -rn "@PreAuthorize\|hasRole\|isAuthenticated\|SecurityContext" \
    BOOT-INF/classes/ --include="*.java"

# Find endpoints that DON'T require auth (often misconfigured)
grep -rn "permitAll\|anonymous\|antMatchers\|requestMatchers" \
    BOOT-INF/classes/ --include="*.java"
```

**What to look for:** Endpoints in the security config that are in `permitAll()` but do database work. These are unauthenticated injection candidates.

### 5 — Find input sanitization (and how to bypass it)

```bash
# Find regex-based filters
grep -rn "replaceAll\|matches\|Pattern\|regex" \
    BOOT-INF/classes/ --include="*.java"

# Find explicit allow/deny lists
grep -rn "contains\|blacklist\|whitelist\|filter\|sanitize" \
    BOOT-INF/classes/ --include="*.java" -i
```

**What to look for:** The difference between `Matcher.matches()` (matches entire string) and `Matcher.find()` (matches substring). `matches()` is often bypassed by constructing a value that satisfies the pattern structurally while still containing an injection payload.

### 6 — Map data flow for second-order injection

```bash
# Find UPDATE statements that write user-controlled data
grep -rn "UPDATE\|INSERT" BOOT-INF/classes/ --include="*.java" -A5

# Cross-reference: find SELECT statements that read the same columns
grep -rn "SELECT" BOOT-INF/classes/ --include="*.java" -A5

# Find profile edit / settings update endpoints (classic second-order sources)
grep -rn "edit\|update\|profile\|settings\|preferences" \
    BOOT-INF/classes/ --include="*.java" -i -l
```

Second-order injection: the payload is written through one endpoint (safe at write time) and executed when retrieved by a different endpoint (unsafe at read time). Look for columns that are written by one controller and read by another.

### 7 — Find file operation and command execution vectors

```bash
# PostgreSQL file ops (COPY TO/FROM)
grep -rn "COPY\|lo_export\|lo_create\|large.object" \
    BOOT-INF/classes/ --include="*.java" -i

# OS command execution
grep -rn "Runtime.exec\|ProcessBuilder\|exec(" \
    BOOT-INF/classes/ --include="*.java"

# File system access
grep -rn "FileInputStream\|FileOutputStream\|new File(" \
    BOOT-INF/classes/ --include="*.java"
```

### Summary grep — run all at once

```bash
cd BOOT-INF/classes/

echo "=== SECRETS ===" && \
grep -rn "secret\|jwtSecret\|password\|api.key" . --include="*.properties" --include="*.yml" -i

echo "=== SQL SINKS ===" && \
grep -rn '".*SELECT\|".*INSERT\|".*UPDATE\|".*WHERE' . --include="*.java"

echo "=== UNAUTHENTICATED ROUTES ===" && \
grep -rn "permitAll\|antMatchers\|requestMatchers" . --include="*.java"

echo "=== INPUT FILTERS ===" && \
grep -rn "replaceAll\|Matcher\|Pattern.compile" . --include="*.java"

echo "=== FILE/CMD OPS ===" && \
grep -rn "Runtime.exec\|COPY\|lo_export" . --include="*.java" -i
```

---

## Gotchas & Notes

- **Always set `JAVA_HOME` when running fernflower.jar** — if your system default is Java 25 EA, the jar itself may fail to run or produce garbled output. Use `JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 java -jar fernflower.jar ...`
- **Spring Boot fat JARs need two steps**: Fernflower decompiles to a new JAR; `jar -xf` extracts the source files from that JAR. Both steps are required.
- **Fernflower cannot decompile obfuscated code faithfully** — obfuscated method names (`a()`, `b()`) survive decompilation but are unreadable. Use `jadx` for obfuscated Android APKs, which has better renaming heuristics.
- **The `lib/` directory inside Spring Boot JARs contains third-party dependencies** — these are usually noise. Focus enumeration on `BOOT-INF/classes/com/` (the application's own package).
- **`application.properties` is plain text inside the JAR** — you don't need to fully decompile to read it: `jar -xf <jar> && cat BOOT-INF/classes/application.properties`
- **The September 2025 Fernflower update changed the required JDK from 17 to 21** — modules cloned before that date used Java 17. If you see `error: invalid source release: 17` during the build, `git checkout c03cae7` reverts to the pre-update state (then use `msopenjdk-17` or `msopenjdk-21`).

---

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — full methodology for finding SQLi via source review
- [[attack/web/advanced_sqli_character_bypasses]] — exploiting what Fernflower helps you find
- [[attack/web/second_order_sqli]] — tracing source/sink pairs in decompiled code
- [[attack/web/error_based_sqli]] — exploiting exception handling found via source review
- [[tools/utility/psql]] — following up on database credentials found in application.properties
- [[labs/htb/advanced_sql_injections/decompiling_java_archives]] — lab applying this workflow

## Sources

- raw/advanced_sql_injections/identifying_vulnerabilities_decompiling_java_archives.md
