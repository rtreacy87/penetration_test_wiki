---
tags: [attack, attack/privesc]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 3
---

# Linux Privilege Escalation — Enumeration

Post-exploitation enumeration checklist for Linux: what to look for and how to find it before attempting any escalation.

## Overview

Enumeration is the foundation of successful privilege escalation. Running [[tools/linpeas]] early is recommended, but manual enumeration builds the knowledge needed when tools can't be transferred or when they miss nuanced misconfigurations. This page covers environment, users, services, internals, credentials, and cron — the full situational awareness picture.

## Key Concepts / Techniques

### Situational Awareness (First 60 Seconds)

```bash
whoami
id
hostname
ip a                  # or: ifconfig
sudo -l               # check immediately — may need no password
cat /etc/os-release
uname -a              # kernel version
echo $PATH
env                   # full environment — look for embedded passwords
```

### OS and Kernel

```bash
cat /etc/os-release
cat /proc/version
uname -a
lscpu
cat /etc/shells       # note if tmux/screen are valid login shells
```

The kernel version is critical for identifying CVE candidates. Use `uname -a` output to search for public exploits. Bash version below 4.1 may be vulnerable to Shellshock.

### Users and Groups

```bash
cat /etc/passwd                    # all users, shells, home dirs
cat /etc/passwd | cut -f1 -d:     # usernames only
grep "sh$" /etc/passwd             # users with login shells
cat /etc/group                     # all groups and members
getent group sudo                  # who is in the sudo group
getent group docker                # high-value group membership
getent group lxd
getent group disk
lastlog                            # last login times per user
w                                  # currently logged-in users
```

Password hashes may appear directly in `/etc/passwd` on embedded/legacy systems — extract and crack offline. Hash prefixes: `$1$` = MD5, `$5$` = SHA-256, `$6$` = SHA-512, `$2a$` = BCrypt.

### Home Directories and SSH Keys

```bash
ls /home
# For each home dir:
ls -la /home/<user>/
cat /home/<user>/.bash_history
ls /home/<user>/.ssh/
# Check known_hosts for pivot targets
cat /home/<user>/.ssh/known_hosts
```

SSH private keys found for other users can enable lateral movement or privilege escalation. If a privileged user's key is readable, connect back as them.

### Network and Routing

```bash
ip a
cat /etc/hosts                     # internal hostnames — AD pivot clues
cat /etc/resolv.conf               # internal DNS server
route                              # or: netstat -rn
arp -a                             # recently communicated hosts
```

Additional NICs indicate multi-subnet environments and potential pivot targets.

### File Systems

```bash
lsblk                              # block devices
df -h                              # mounted filesystems
cat /etc/fstab | grep -v "#"      # check for creds in mount options
cat /etc/fstab | grep -v "#" | column -t
# Look for unmounted partitions
```

Search for credentials in fstab options (`password=`, `credentials=`). Unmounted volumes may contain sensitive data accessible after privilege escalation.

### Hidden Files and Temporary Directories

```bash
# Hidden files owned by current user
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep $USER

# Hidden directories
find / -type d -name ".*" -ls 2>/dev/null

# Temporary files — check for scripts, configs, keys
ls -la /tmp /var/tmp /dev/shm
```

`/tmp` is cleared on reboot; `/var/tmp` retains files up to 30 days.

### Running Services and Processes

```bash
# Services running as root — high-value targets
ps aux | grep root

# All running services
ps aux

# Installed packages (Debian/Ubuntu)
apt list --installed 2>/dev/null | tr "/" " " | cut -d" " -f1,3

# Sudo version — check for CVE-2021-3156
sudo -V | head -1

# Binaries in standard locations
ls -l /bin /usr/bin /usr/sbin
```

### Services, Configs, and Scripts

```bash
# Configuration files
find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null

# Shell scripts
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

# Running services by user
ps aux | grep root
```

Look for world-writable scripts that execute as root. Compare installed binaries against GTFOBins.

### Internals and procfs

```bash
# Command lines of running processes (may expose passwords)
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"

# Trace system calls (useful for finding credentials passed on CLI)
strace <program>
```

### Cron Jobs

```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.hourly/ /etc/cron.d/
cat /etc/cron.d/*

# World-writable files (scripts that may be run by cron)
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null

# Watch for cron execution in real time
# Use pspy — see [[tools/pspy]]
```

Look for patterns in file modification timestamps to infer cron schedules even when the crontab is not readable.

### Credential Hunting

```bash
# WordPress/web app DB credentials
grep 'DB_USER\|DB_PASSWORD' /var/www/html/wp-config.php 2>/dev/null

# Config files containing "password", "credential", "username"
find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null

# History files
find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null
history

# SSH keys
ls ~/.ssh/
find / -name "id_rsa" -o -name "id_ed25519" 2>/dev/null

# .bak files, .xml, .conf — grep for passwords
find / -name "*.bak" -o -name "*.xml" -o -name "*.conf" 2>/dev/null | \
  xargs grep -l "password" 2>/dev/null
```

Try every collected credential against all users on the system — password reuse is common.

## Commands / Syntax

GTFOBins cross-reference — compare installed binaries to the GTFOBins API:

```bash
for i in $(curl -s https://gtfobins.github.io/api.json | jq -r '.executables | keys[]'); do
    if grep -q "$i" installed_pkgs.list; then
        echo "Check for GTFO: $i"
    fi
done
```

Strace for credential exposure:

```bash
strace -p <PID> 2>&1 | grep -i "pass\|key\|token"
```

## Flags & Options

| Command | Key Flag | Purpose |
|---|---|---|
| `find` | `-perm -o+w` | World-writable files |
| `find` | `-perm -4000` | SUID files |
| `find` | `-perm -2000` | SGID files |
| `uname` | `-a` | All kernel/OS info |
| `lastlog` | (none) | Last login per user |
| `ps aux` | (none) | All processes with user |
| `getent` | `group <name>` | Members of a group |

## Gotchas & Notes

- `sudo -l` sometimes works without a password (NOPASSWD entries) — always try it first
- `/etc/passwd` hashes are rare but real on embedded/legacy systems
- Bash version 4.1 and below is vulnerable to Shellshock (`CVE-2014-6271`)
- `tmux` and `screen` appearing in `/etc/shells` indicates multiplexers are available for session hijacking
- Internal DNS (`/etc/resolv.conf`) pointing at an AD server is valuable pivot context
- Credentials in `wp-config.php`, `.env`, and similar web files are frequently valid for SSH or database access
- `lastlog` timestamps reveal active vs. stale accounts — stale accounts may have weaker credentials

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_sudo_suid]]
- [[attack/linux_privesc_services]]
- [[attack/linux_privesc_kernel]]
- [[tools/linpeas]]
- [[tools/pspy]]

## Sources

- raw/linux_privilege_escalation/information_gathering_environment_enumeration.md
- raw/linux_privilege_escalation/information_gathering_linux_services_and_internals_enumeration.md
- raw/linux_privilege_escalation/information_gathering_credential_hunting.md
