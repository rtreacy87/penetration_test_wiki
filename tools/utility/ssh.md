---
tags: [tool, reference]
module: footprinting
last_updated: 2026-05-12
source_count: 1
---

# SSH (Secure Shell)

SSH client reference: connecting with passwords and keys, creating and deleting key pairs, and cleaning up known_hosts fingerprints left behind by lab VMs.

## Overview

SSH is the standard protocol for remote terminal access on Linux/Unix systems. From a pentesting workflow perspective, two tasks come up constantly:

1. **Connecting** — password or key-based, sometimes with a non-standard port or specific key file
2. **Fingerprint management** — every new SSH connection writes a fingerprint to `~/.ssh/known_hosts`. When you spin up VMs repeatedly (HTB, THM, lab ranges), the same IPs rotate through different servers and the stale fingerprints cause connection warnings or outright failures. Cleaning them out is routine maintenance.

See [[enumeration/linux_remote_mgmt]] for SSH enumeration, banner grabbing, and identifying misconfigurations.

## Installation

```bash
# Check if installed
ssh -V 2>&1 | head -1 || echo "not installed"

# Install (Kali / Parrot — the OpenSSH client is pre-installed)
sudo apt install openssh-client -y

# Verify
ssh -V
```

---

## Connecting

### Password authentication

```bash
ssh user@10.129.x.x
```

SSH will prompt for the password interactively. To specify the port if it is not 22:

```bash
ssh -p 2222 user@10.129.x.x
```

To pass the password non-interactively (scripts only — avoid in interactive use):

```bash
sshpass -p 'MyPassword!' ssh user@10.129.x.x
# Install sshpass if needed: sudo apt install sshpass -y
```

### Key-based authentication

```bash
ssh -i ~/.ssh/id_ed25519 user@10.129.x.x
```

| Part | What it does |
|------|-------------|
| `-i ~/.ssh/id_ed25519` | Path to the **private** key file. The matching public key must be in `~/.ssh/authorized_keys` on the server. |
| `user@host` | Username and target host or IP |

If the key has a passphrase, SSH prompts for it once per session (or once per `ssh-agent` load).

Combine port and key:

```bash
ssh -i ~/.ssh/id_rsa -p 2222 user@10.129.x.x
```

---

## Creating a Key Pair

SSH key pairs consist of a **private key** (stays on your machine, never shared) and a **public key** (placed on the server).

### Recommended: Ed25519 (modern, shorter, faster)

```bash
ssh-keygen -t ed25519 -C "htb-labs"
```

### Alternative: RSA 4096 (wider compatibility with older systems)

```bash
ssh-keygen -t rsa -b 4096 -C "htb-labs"
```

Both commands walk you through the same prompts:

```
Enter file in which to save the key (~/.ssh/id_ed25519):   ← press Enter for default, or name it
Enter passphrase (empty for no passphrase):                 ← passphrase encrypts the private key on disk
Enter same passphrase again:
```

For lab use, an empty passphrase is fine — you do not need to re-enter it on every connection.

This creates two files:

| File | Purpose |
|------|---------|
| `~/.ssh/id_ed25519` | **Private key** — never share this |
| `~/.ssh/id_ed25519.pub` | **Public key** — copy this to the server |

### Copy the public key to a server

```bash
# Automated (if you have password access first)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@10.129.x.x

# Manual (paste into authorized_keys on the server)
cat ~/.ssh/id_ed25519.pub
# Then on the server: echo "<paste>" >> ~/.ssh/authorized_keys
```

### Create a key with a custom name (for lab-specific keys)

```bash
ssh-keygen -t ed25519 -f ~/.ssh/htb_ed25519 -C "htb-labs"
ssh -i ~/.ssh/htb_ed25519 user@10.129.x.x
```

---

## Deleting a Key Pair

Delete both the private and public key files. There is no central registry — just remove the files:

```bash
# Remove a specific key pair
rm ~/.ssh/id_ed25519 ~/.ssh/id_ed25519.pub

# Or for a custom-named key
rm ~/.ssh/htb_ed25519 ~/.ssh/htb_ed25519.pub
```

If you delete a key that is still in a server's `authorized_keys`, the server will reject future key-based logins. You would need to reconnect via password and remove the corresponding public key from `~/.ssh/authorized_keys` on the server.

---

## Managing known_hosts — Cleaning Up VM Fingerprints

Every time you SSH to a new host, the server's public key fingerprint is saved to `~/.ssh/known_hosts`. On next connection, SSH checks the fingerprint matches. If it does not (because the VM was rebuilt and has a new key), you see:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

This is the most common friction point when cycling through HTB/lab VMs that reuse IPs.

### Remove a single host's fingerprint

```bash
ssh-keygen -R 10.129.x.x
```

This finds all entries for that IP in `~/.ssh/known_hosts` and removes them. Run this whenever a VM is reset or a new box is spawned at the same IP.

```bash
# Also works with hostnames
ssh-keygen -R target.htb
```

### Remove all entries in a lab IP range

When you finish a session and want to clear everything in the HTB VPN range (10.129.0.0/16):

```bash
sed -i '/^10\.129\./d' ~/.ssh/known_hosts
```

For the 10.10.x.x range (THM or other labs):

```bash
sed -i '/^10\.10\./d' ~/.ssh/known_hosts
```

### Clear the entire known_hosts file

```bash
> ~/.ssh/known_hosts
# or
truncate -s 0 ~/.ssh/known_hosts
```

This is safe for a lab machine where all saved fingerprints are disposable. Do not do this on a production machine — you lose protection against MITM for all hosts.

### Skip fingerprint checking entirely (throwaway connections)

For one-off connections where you do not want any fingerprint saved:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@10.129.x.x
```

| Option | What it does |
|--------|-------------|
| `StrictHostKeyChecking=no` | Accept any host key without prompting, even if it has changed |
| `UserKnownHostsFile=/dev/null` | Do not read or write any fingerprints — the connection leaves no trace in known_hosts |

### Useful aliases for lab workflow

Add these to `~/.bashrc` or `~/.zshrc`:

```bash
# Remove a single IP fingerprint quickly
alias sshclean='ssh-keygen -R'
# Usage: sshclean 10.129.x.x

# Clear all HTB VPN fingerprints
alias sshclean-htb='sed -i "/^10\.129\./d" ~/.ssh/known_hosts && echo "HTB fingerprints cleared"'

# Clear all THM fingerprints
alias sshclean-thm='sed -i "/^10\.10\./d" ~/.ssh/known_hosts && echo "THM fingerprints cleared"'

# SSH without saving fingerprints (for lab boxes)
alias sshlab='ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'
# Usage: sshlab user@10.129.x.x
```

---

## SSH Config File — Per-Host Settings

`~/.ssh/config` lets you define per-host defaults so you do not have to type flags every connection.

```
# Lab shortcut — no fingerprint saving, custom key
Host htb
    HostName 10.129.x.x
    User root
    IdentityFile ~/.ssh/htb_ed25519
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

# Specific box shortcut
Host footprinting-hard
    HostName 10.129.202.41
    User htb-student
    IdentityFile ~/.ssh/id_ed25519
    Port 22
```

With the above, `ssh htb` connects using all the defined options. Update `HostName` each time you get a new target IP.

Permissions on the config file must be restricted or SSH refuses to use it:

```bash
chmod 600 ~/.ssh/config
```

---

## Flags & Options

| Flag | Description |
|------|-------------|
| `-i <file>` | Identity file (private key) |
| `-p <port>` | Non-standard port |
| `-l <user>` | Username (alternative to `user@host`) |
| `-v` / `-vv` / `-vvv` | Verbose output — use to diagnose auth failures |
| `-A` | Forward the local SSH agent — lets you hop to other hosts using your local key |
| `-L <local_port>:<host>:<remote_port>` | Local port forward (tunnel a remote service to localhost) |
| `-R <remote_port>:<host>:<local_port>` | Remote port forward |
| `-D <port>` | SOCKS5 proxy — tunnel all traffic through the SSH connection |
| `-N` | No remote command — used with tunnels (`-L`, `-R`, `-D`) to keep the connection open without a shell |
| `-o <option>=<value>` | Pass a config option inline (e.g., `-o StrictHostKeyChecking=no`) |

## Gotchas & Notes

- Private key files must have `600` permissions — SSH refuses to use keys that are world-readable: `chmod 600 ~/.ssh/id_ed25519`
- The `known_hosts` file can hold multiple entries per IP (if the server ever changed its key). `ssh-keygen -R` removes all of them at once
- `StrictHostKeyChecking=no` with `UserKnownHostsFile=/dev/null` is safe for lab throwaway connections but is a MITM risk on untrusted networks — never use it when connecting to anything you care about
- If key auth fails but the key exists, check: correct `-i` path, key permissions (`600`), and whether the public key is actually in `~/.ssh/authorized_keys` on the server (use `-v` to see which auth methods were tried)
- `ssh-copy-id` appends to `authorized_keys` — it does not replace it. Run it multiple times and you get duplicate entries (harmless but untidy)
- Ed25519 keys are not supported on very old OpenSSH versions (pre-6.5, circa 2014). RSA 4096 is the safe fallback for legacy servers

## Related Pages

- [[enumeration/linux_remote_mgmt]] — SSH enumeration, dangerous settings, banner grabbing, ssh-audit
- [[tools/utility/xfreerdp]] — RDP equivalent for Windows targets
- [[shell_commands/bash]] — reverse shells, file transfer over SSH, tunnelling one-liners

## Sources

- raw/footprinting/linux_remote_management_protocols.md
