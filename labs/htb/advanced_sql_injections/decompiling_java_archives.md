---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Decompiling Java Archives

Target access: SSH to the target with `student:academy.hackthebox.com` and SCP the JAR from `/opt/bluebird`.

This lab involves decompiling a Java JAR to read source code. The walkthrough uses **Fernflower** (CLI) as the primary approach and notes JD-GUI as a GUI alternative — Fernflower is the practical choice for any headless/SSH environment.

## Tool Installation

See [[tools/enumeration/fernflower]] for the full installation guide, build troubleshooting, and systematic JAR enumeration methodology.

**Verified quick-start on Kali rolling (Java 25 default):**

```bash
# Install Microsoft's stable Java 21 JDK (openjdk-21-jdk in Kali repos is EA and breaks the build)
sudo apt install msopenjdk-21 -y

# Clone and build
git clone https://github.com/fesh0r/fernflower.git
cd fernflower/
JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 ./gradlew build -x test --no-configuration-cache
# Output: build/libs/fernflower.jar
```

**If Fernflower won't build, jadx is a full substitute:**
```bash
sudo apt install jadx -y
```

---

## Question 1 — Value of `bluebird.app.jwtSecret` in application.properties

**Goal:** Decompile the BlueBird JAR file, locate `application.properties`, and read the JWT secret.

**Why start with decompilation:** You've been given or discovered a running Java application. Before probing it with HTTP requests or fuzzing inputs, source code review is the highest-value first step in a white-box context. It tells you every endpoint that exists (not just the ones the UI exposes), every SQL query being built, every secret hardcoded in config — all without generating any server-side log entries. A JWT secret in `application.properties` lets you forge authentication tokens, which may grant access to the entire application without needing to crack a password or exploit a vulnerability.

**Why not just use strings or hexdump:** A JAR is a ZIP archive containing compiled `.class` bytecode, not plaintext. `strings` would find some hardcoded literals but would miss anything computed at runtime and would give you no structural understanding of the code. Decompilation reconstructs the actual Java source, which is readable and greppable.

### CLI path (Fernflower + grep)

**Step 1 — Download the JAR from the target**

```bash
scp -r student@<TARGET_IP>:/opt/bluebird ./
```

You're copying the whole `/opt/bluebird` directory rather than just the JAR because the directory also contains PostgreSQL log files from a running instance. Those logs may reveal query patterns and database usernames — worth having even if you don't need them immediately.

**Step 2 — Decompile with Fernflower**

```bash
mkdir BlueBirdSourceCode/
java -jar fernflower/build/libs/fernflower.jar \
    ./bluebird/BlueBird-0.0.1-SNAPSHOT.jar \
    BlueBirdSourceCode/
```

Fernflower writes a new JAR to `BlueBirdSourceCode/` — it doesn't produce loose `.java` files directly. That JAR contains the decompiled source. To work with the individual files, you need to extract it.

**Why Fernflower over JD-GUI:** JD-GUI is a desktop application requiring a display. Fernflower runs as a CLI jar — works over SSH, in CI pipelines, or anywhere Java runs. They produce comparable output; Fernflower is actively maintained and handles obfuscated code better.

**Step 3 — Extract the decompiled archive**

```bash
cd BlueBirdSourceCode/
jar -xf BlueBird-0.0.1-SNAPSHOT.jar
```

Spring Boot JARs are "fat JARs" — they contain the application class files nested under `BOOT-INF/classes/`. After extraction, the structure looks like:

```
BOOT-INF/
  classes/          ← application source (controllers, models, config)
  lib/              ← bundled dependencies
META-INF/
org/springframework/ ← embedded Spring Boot launcher
```

**Why this matters:** You only need to look at `BOOT-INF/classes/` — the rest is framework code you didn't write and can't easily modify.

**Step 4 — Locate application.properties**

The fastest way to find config files without browsing the tree manually:

```bash
find BOOT-INF/classes -name "application.properties"
# Expected: BOOT-INF/classes/application.properties
```

At this stage you might also note other interesting files:
- `application-prod.properties` — production overrides, often contains database passwords
- `*Controller.java` files — these define every HTTP endpoint and are the primary SQLi hunting ground

**Step 5 — Read the JWT secret**

```bash
grep "jwtSecret" BOOT-INF/classes/application.properties
```

The value of `bluebird.app.jwtSecret` is returned directly. With this key, you can:
- Decode any JWT the application issues (using jwt.io or PyJWT)
- Forge a JWT for any user, including admin accounts that may not have a self-registration path

**Why this is the answer the question is looking for, and what to do with it:** The JWT secret is the root signing key. Forged tokens bypass authentication entirely — no password cracking, no brute force, no SQL injection required at the authentication layer. Always grep for `secret`, `password`, `key`, `token`, `credential` in application config files.

### GUI alternative evaluation

VS Code (`code ./BOOT-INF/classes/`) provides a tree browser and syntax highlighting. **This is not practical over SSH** — it requires a desktop session or X forwarding. Fernflower + grep produces identical results and works in any terminal.

**Verdict: Fernflower CLI is the better choice for all pentest environments.**

---

## Lessons Learned

- Spring Boot JARs are nested archives; `jar -xf` unpacks the outer archive to reveal `BOOT-INF/classes/` where application config and compiled `.class` files live
- `application.properties` often contains hardcoded secrets (JWT keys, database passwords, API tokens) — always grep for them after decompilation; a JWT secret can be more valuable than any individual SQLi
- The decompiled source is now locally greppable — all subsequent labs (hunting for SQL errors, searching for strings, reading the error-based SQLi vulnerability) build on having this source available

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — full white-box source review methodology
- [[tools/utility/psql]] — PostgreSQL CLI for following up on discovered credentials
