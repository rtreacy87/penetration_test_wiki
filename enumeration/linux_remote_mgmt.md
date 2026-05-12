---
tags: [enumeration, protocol]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Linux Remote Management Protocols

SSH, Rsync, and R-services (rlogin, rsh, rexec, rwho): enumeration, misconfigurations, and attack vectors.

## Overview

Linux servers expose remote management through SSH (the standard), legacy R-services (deprecated but still found), and Rsync (file synchronization, often accessible without auth). These services are prime targets because:
- SSH may have weak/default credentials or misconfigured auth methods
- R-services use trust-based auth via `.rhosts` and `hosts.equiv` files — trivially bypassed when misconfigured
- Rsync shares sometimes contain sensitive files and are accessible without authentication

## Key Concepts / Techniques

### SSH (Secure Shell) — TCP 22

OpenSSH is the standard implementation on Linux. Six authentication methods:
1. Password authentication
2. Public-key authentication
3. Host-based authentication
4. Keyboard authentication
5. Challenge-response authentication
6. GSSAPI authentication

**Dangerous SSH Settings:**

| Setting | Risk |
|---------|------|
| `PasswordAuthentication yes` | Enables brute-force |
| `PermitEmptyPasswords yes` | Allows empty passwords |
| `PermitRootLogin yes` | Direct root login |
| `Protocol 1` | Vulnerable to MITM |
| `X11Forwarding yes` | Historical CVE vector |
| `AllowTcpForwarding yes` | Tunneling/pivoting |
| `PermitTunnel` | VPN-like tunneling |

**SSH version banner format:** `SSH-2.0-OpenSSH_8.2p1` — the version number reveals if the server is vulnerable to known CVEs.

### Rsync — TCP 873

Rsync is a file synchronization tool that uses a delta-transfer algorithm. It can run as a standalone daemon (accessible without SSH) or over SSH.

When running as a daemon (without SSH), Rsync may expose shares without authentication. The `@RSYNCD:` banner confirms a standalone daemon. Directories listed can then be enumerated and downloaded.

**Key Rsync attack path:** Find the daemon running unauthenticated → list shares → look for `.ssh/` directories with private keys or config files with credentials.

### R-Services — TCP 512, 513, 514

Legacy Unix remote access suite. No encryption. Trust is established via `/etc/hosts.equiv` (global) and `~/.rhosts` (per-user). These files list trusted host/user pairs.

| Service | Port | Command | Description |
|---------|------|---------|-------------|
| rexec | 512 | `rexec` | Execute shell commands remotely |
| rlogin | 513 | `rlogin` | Remote login |
| rsh | 514 | `rsh` | Remote shell without login |
| rwho | UDP 513 | `rwho` | List logged-in users on network |
| rusers | — | `rusers` | Detailed logged-in user info |

**`.rhosts` wildcards**: The `+` character means "any". A `.rhosts` entry of `+ +` trusts any user from any host — complete bypass.

## Commands / Syntax

```bash
# SSH fingerprinting with ssh-audit
git clone https://github.com/jtesta/ssh-audit.git && cd ssh-audit
./ssh-audit.py 10.129.14.132

# SSH banner grab
nc -nv 10.129.14.132 22

# SSH verbose connection (reveals auth methods)
ssh -v cry0l1t3@10.129.14.132

# Force password authentication (useful for brute-forcing)
ssh -v cry0l1t3@10.129.14.132 -o PreferredAuthentications=password

# Rsync — scan
sudo nmap -sV -p873 127.0.0.1

# Rsync — list shares (via netcat)
nc -nv 127.0.0.1 873
@RSYNCD: 31.0
#list

# Rsync — enumerate a specific share
rsync -av --list-only rsync://127.0.0.1/dev

# Rsync — download all files from share
rsync -av rsync://127.0.0.1/dev ./local-copy/

# Rsync over SSH
rsync -av -e ssh rsync://user@10.129.14.128/dev ./local-copy/
rsync -av -e "ssh -p2222" rsync://user@10.129.14.128/dev ./local-copy/

# R-services — scan
sudo nmap -sV -p512,513,514 10.0.17.2

# rlogin (if .rhosts allows)
rlogin 10.0.17.2 -l htb-student

# rwho — list logged-in users
rwho

# rusers — detailed logged-in user info
rusers -al 10.0.17.5
```

## Flags & Options

### ssh-audit Output Flags

| Flag (in output) | Meaning |
|-----------------|---------|
| `[fail]` | Weak algorithm; should be removed |
| `[warn]` | Caution; may be exploitable |
| `[info]` | Informational only |

### Rsync Flags

| Flag | Description |
|------|-------------|
| `-a` | Archive mode (preserves permissions, timestamps) |
| `-v` | Verbose |
| `--list-only` | List files without downloading |
| `-e ssh` | Use SSH transport |

## Version Detection & Exploit Research

SSH banners are the most reliable unauthenticated version source on Linux systems — the OpenSSH version string is transmitted in plaintext before any key exchange. Rsync daemon banners similarly identify protocol versions. R-services have no meaningful versioning but their presence alone is a finding.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| SSH banner | `nc -nv <IP> 22` | `SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.5` — full version + OS |
| Nmap service | `nmap -sV -p22 <IP>` | OpenSSH version + OS hints |
| ssh-audit | `ssh-audit.py <IP>` | Algorithm support, known-weak ciphers, CVE flags |
| Rsync banner | `nc -nv <IP> 873` | `@RSYNCD: 31.0` — protocol version |
| Nmap SSH scripts | `nmap -p22 --script ssh2-enum-algos,ssh-auth-methods <IP>` | Supported algorithms + enabled auth methods |

**Parse the SSH banner:** `SSH-<protocol>-<software>_<version> <OS-info>`
- `SSH-2.0-OpenSSH_7.2p1` → look for CVEs affecting OpenSSH 7.2.x
- `SSH-2.0-OpenSSH_8.9p1` → relatively recent; still check release notes
- `SSH-1.99-OpenSSH_...` → server accepts SSHv1 — flag this

### Searching for Exploits

```bash
# Searchsploit
searchsploit openssh
searchsploit openssh 7.2    # version-specific
searchsploit rsync

# Metasploit
msf6> search type:exploit name:openssh
msf6> search type:auxiliary name:ssh

# CVE lookup by version
# https://nvd.nist.gov/products/cpe/search — search for cpe:openssh
```

### Notable CVEs

| CVE | Affected Versions | Impact |
|-----|------------------|--------|
| CVE-2024-6387 (regreSSHion) | OpenSSH < 4.4p1 and 8.5p1–9.8p1 | Signal handler race condition — unauthenticated RCE as root |
| CVE-2023-38408 | OpenSSH < 9.3p2 | ssh-agent remote code execution via forwarded agent |
| CVE-2016-0777 | OpenSSH 5.4–7.1 | Roaming feature leaks private key material to attacker-controlled server |
| CVE-2016-3115 | OpenSSH 7.2p1 | X11 forwarding command injection |
| CVE-2018-15473 | OpenSSH < 7.7 | Username enumeration via timing side-channel |
| CVE-2021-41617 | OpenSSH < 8.8 | Privilege escalation via supplemental group inheritance |

## Gotchas & Notes

- **SSH-1 vs SSH-2**: If the banner shows `SSH-1.99`, the server accepts both protocol versions. SSH-1 is vulnerable to MITM. Flag this.
- **X11Forwarding CVE-2016-3115**: Affected OpenSSH 7.2p1. Check the version from the banner.
- **`PreferredAuthentications=password`**: Forces the client to use password auth even if key auth is also available. Use this to attempt brute-forcing when keys are accepted.
- **Rsync `.ssh/` directories**: A common finding — developers rsync their home directory including `~/.ssh/` to a backup share accessible without authentication.
- **R-services in the wild**: Rare but present in Solaris, HP-UX, AIX environments, and sometimes misconfigured Linux. Always scan 512, 513, 514 during internal assessments.
- **`hosts.equiv` is global**: Any host listed in `/etc/hosts.equiv` can log in as any user without a password. An entry of just `+` means any host, any user.
- **`.rhosts` with `+` entries**: A `.rhosts` file containing `+ +` grants unauthenticated root-equivalent access. This is the R-services equivalent of `guest ok = yes`.
- **rwho broadcasts**: The rwho daemon periodically broadcasts user session info over UDP. Capture this passively on a local network segment to enumerate active users.

## Related Pages

- [[enumeration/_overview]]
- [[enumeration/windows_remote_mgmt]]
- [[tools/utility/ssh]] — SSH client reference: connecting, key management, known_hosts cleanup

## Sources

- raw/footprinting/linux_remote_management_protocols.md
