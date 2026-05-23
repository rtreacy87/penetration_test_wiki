---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Searching for Strings

Target access: SSH to the target with `student:academy.hackthebox.com` and SCP the JAR from `/opt/bluebird`. Requires the Fernflower decompilation setup from [[labs/htb/advanced_sql_injections/decompiling_java_archives]].

This lab is fully CLI-based using Fernflower and grep. VS Code is noted as a GUI browsing option but is not required.

## Tool Installation

See [[labs/htb/advanced_sql_injections/decompiling_java_archives]] for the full setup and troubleshooting. Verified working on Kali rolling (Java 25 default):

**jadx (no Java setup required):**
```bash
sudo apt install jadx -y
jadx -d BlueBirdSourceCode/ bluebird/BlueBird-0.0.1-SNAPSHOT.jar
```

**Fernflower (verified working with `msopenjdk-21`):**
```bash
sudo apt install msopenjdk-21 -y
git clone https://github.com/fesh0r/fernflower.git
cd fernflower/
JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 ./gradlew build -x test --no-configuration-cache
cd ..
```

---

## Question 1 — Which variable in the INSERT query cannot be exploited?

**Goal:** Decompile the BlueBird JAR, find all INSERT statements in the source, and identify which variable in an INSERT concatenation cannot be exploited for SQL injection.

**Why look for INSERT statements specifically:** SQL injection in SELECT queries gets the most attention, but INSERT, UPDATE, and DELETE statements are equally worth auditing. An injectable INSERT can write arbitrary data to the database — and if stacked queries are supported, it can execute additional SQL beyond the INSERT itself. The question frames this as finding the *un*-exploitable variable, which teaches a critical skill: not every concatenated value is a usable injection point, even if it looks like one at first glance.

**Step 1 — Decompile and extract (if not already done)**

```bash
scp -r student@<TARGET_IP>:/opt/bluebird ./
mkdir BlueBirdSourceCode/
java -jar fernflower/build/libs/fernflower.jar \
    ./bluebird/BlueBird-0.0.1-SNAPSHOT.jar \
    BlueBirdSourceCode/
cd BlueBirdSourceCode/
jar -xf BlueBird-0.0.1-SNAPSHOT.jar
```

**Step 2 — Search for INSERT statements**

Rather than opening files one by one in a text editor, grep for the SQL keyword directly in the decompiled source:

```bash
grep -rn "INSERT" BOOT-INF/classes/ --include="*.java"
```

The `-n` flag shows line numbers so you can immediately jump to the relevant code. The `--include="*.java"` filter is important — without it, grep will also search `.class` binary files and produce garbage output.

This finds the INSERT statement in `AuthController.java`. Write down the file path and line number — you'll read a narrow section of the file rather than the whole thing.

**Why grep before reading:** A real codebase might have hundreds of Java files. Reading them all in a text editor to find SQL queries would take hours. Grep reduces a 45MB JAR to three or four lines of output in seconds.

**Step 3 — Read the relevant controller section**

```bash
find BOOT-INF/classes -name "AuthController.java"
grep -A 30 "INSERT" BOOT-INF/classes/com/bluebird/controllers/AuthController.java
```

The `-A 30` flag shows 30 lines after the match, giving you enough context to see the full method.

You'll see the `signupPOST()` method. It contains multiple SQL queries. The key observation is that some of them are parameterized (using `?` placeholders passed through JDBC's `PreparedStatement`) while the final INSERT is not — it builds the query by string concatenation. Note which POST parameters feed directly into the string: `name`, `username`, `email`, and `password`.

**Step 4 — Identify which variable cannot be exploited**

Look at how each value is prepared before insertion. For most variables, the raw POST parameter value lands directly in the query string. But trace `password`:

```java
String passwordHash = encoder.encode(password);
```

`encoder.encode()` is BCrypt. Whatever value you supply for `password`, it gets hashed through BCrypt before it touches the SQL query. BCrypt output is always a 60-character string in the format `$2b$12$[A-Za-z0-9./]{53}` — a fixed structure using only characters that have no special meaning in SQL.

**Why BCrypt is an effective (if accidental) filter:** You might try to inject SQL through the password field by submitting something like `' OR 1=1--`. After BCrypt encoding, this becomes something like `$2b$12$xK9mU...rQn9z` — a harmless random-looking string. Your SQL metacharacters are gone. The encoding acts as a transformation that strips injection potential, even though it was never designed as a security control for SQLi.

**Why the other variables ARE exploitable:** `name`, `username`, and `email` go into the query as-is, with no transformation. This becomes the attack surface for the [[labs/htb/advanced_sql_injections/reading_and_writing_files]] and [[labs/htb/advanced_sql_injections/command_execution]] labs.

```bash
grep -n "passwordHash\|encoder.encode" BOOT-INF/classes/com/bluebird/controllers/AuthController.java
```

The answer is `passwordHash` — it cannot be exploited because it is the *hash of* the user-supplied value, not the raw value itself.

**Could you exploit it another way:** Theoretically you could try to craft a password whose BCrypt hash happens to contain SQL metacharacters, but BCrypt output is Base64-like (`./A-Za-z0-9`) and `$` is the only special character that appears — and dollar-sign quoting (`$$`) is actually valid PostgreSQL syntax, not an injection character. In practice: no, this is a dead end.

### GUI alternative evaluation

VS Code provides syntax highlighting and allows clicking between files. **For this task, grep is faster** — the codebase is small and the INSERT statement is found in one grep pass. VS Code requires a desktop session or X forwarding.

**Verdict: grep is practical and sufficient. VS Code is optional comfort.**

---

## Lessons Learned

- Not every concatenated variable in an unsafe query is exploitable — always trace the variable back to its assignment and look for transformations (encoding, hashing, type conversion)
- BCrypt, by producing output in a restricted character set, acts as an inadvertent SQLi filter — this is a coincidence of design, not intentional defense
- Grep for INSERT/UPDATE/SELECT/DELETE before reading source files — it pinpoints injection sinks immediately and saves hours compared to file browsing
- The injectable variables in this specific INSERT (`name`, `username`, `email`) are directly exploited in the reading_and_writing_files and command_execution labs

## Related Pages

- [[attack/web/white_box_sqli_methodology]] — systematic source review with grep patterns
- [[attack/web/second_order_sqli]] — tracing source/sink pairs
- [[labs/htb/advanced_sql_injections/decompiling_java_archives]] — decompilation prerequisite
