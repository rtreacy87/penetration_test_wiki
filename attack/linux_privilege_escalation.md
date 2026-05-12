---
tags: [attack, attack/privesc]
module: linux_privilege_escalation
last_updated: 2026-05-10
source_count: 26
---

# Linux Privilege Escalation

Comprehensive reference for escalating from a low-privileged shell to root on Linux systems, covering all major attack categories from enumeration through kernel exploits.

## Overview

Linux privilege escalation (privesc) is the process of gaining elevated access on a system after initial foothold. The goal is almost always to become `root` (UID 0). Success depends heavily on thorough enumeration — you need to understand the environment before exploiting it. Automated tools like [[tools/enumeration/linpeas]] will find many issues quickly, but manual enumeration builds the understanding needed to adapt when tools fail.

The major attack categories, roughly in order of ease and frequency:

| Category | Examples | Risk to System |
|---|---|---|
| Enumeration | Environment, credentials, services | None |
| Sudo abuse | Misconfigured sudoers, GTFOBins | Low |
| SUID/SGID/Capabilities | World-writable SUID, cap_dac_override | Low |
| Cron job abuse | World-writable scripts, PATH injection | Low |
| Service exploitation | Logrotate, vulnerable daemons | Medium |
| Container escape | Docker socket, LXD group | Medium |
| Environment abuse | PATH hijacking, wildcard injection | Low |
| Kernel exploits | DirtyPipe, Netfilter, Polkit | High (can crash) |
| Library hijacking | LD_PRELOAD, shared object hijacking, Python | Low-Medium |

## Methodology

1. Run [[tools/enumeration/linpeas]] immediately on landing — collect output for review
2. Manually enumerate environment (OS, kernel, users, PATH, network)
3. Check sudo rights (`sudo -l`) — often the fastest win
4. Find SUID/SGID binaries and capabilities
5. Enumerate cron jobs with [[tools/enumeration/pspy]]
6. Check group memberships (docker, lxd, disk, adm)
7. Hunt credentials in configs, history, web roots
8. Check running services and installed package versions
9. Consider kernel version against known CVEs
10. Try library hijacking if `LD_PRELOAD` or writable library paths exist

## Attack Categories

### Enumeration
See [[attack/linux_privesc_enumeration]] for the full enumeration checklist: environment, users, services, credentials, network, cron, and internals.

### Sudo Rights Abuse
`sudo -l` is always the first check. Misconfigured entries allow executing GTFOBins commands as root, or abusing tool features (`tcpdump -z`, etc.). See [[attack/linux_privesc_sudo_suid]].

### SUID / SGID Binaries
Binaries with SUID set run as the file owner (often root) regardless of who launches them. GTFOBins documents hundreds of exploitable SUID/SGID paths. See [[attack/linux_privesc_sudo_suid]].

### Linux Capabilities
Fine-grained kernel privileges assigned to binaries. `cap_dac_override` lets a binary ignore file permission checks — combined with `vim` or `python`, this writes to `/etc/passwd` or `/etc/shadow`. See [[attack/linux_privesc_sudo_suid]].

### Privileged Groups
Membership in `docker`, `lxd`, `disk`, or `adm` groups provides escalation paths without any other vulnerability. See [[attack/linux_privesc_services]].

### Cron Job Abuse
World-writable scripts run by root cron jobs are the classic win. [[tools/enumeration/pspy]] reveals cron execution without root access. Wildcard injection via `tar` is another cron vector. See [[attack/linux_privesc_services]].

### Service-Based Escalation
Docker socket access, LXD container escape, Kubernetes kubelet API, logrotate (logrotten exploit), vulnerable service versions (Screen 4.5.0), weak NFS exports, tmux session hijacking. See [[attack/linux_privesc_services]].

### Environment Abuse
- **PATH hijacking**: Place a malicious binary early in PATH or in a directory searched by a SUID binary
- **Wildcard abuse**: `tar *` in cron jobs interprets filenames as flags
- **Restricted shell escape**: Command injection, substitution, chaining in rbash/rksh/rzsh

### Kernel Exploits and Library Hijacking
DirtyPipe (CVE-2022-0847), Netfilter CVEs, Polkit/PwnKit (CVE-2021-4034), sudo heap overflow (CVE-2021-3156), plus LD_PRELOAD abuse, shared object hijacking, and Python library hijacking. See [[attack/linux_privesc_kernel]].

## Quick Wins Checklist

```bash
# Identity
whoami; id; groups

# Sudo rights (fastest win)
sudo -l

# SUID binaries
find / -perm -4000 -type f 2>/dev/null

# Capabilities
find /usr/bin /usr/sbin /usr/local/bin -type f -exec getcap {} \;

# Writable files owned by root
find / -writable -user root -type f 2>/dev/null

# Cron jobs
cat /etc/crontab; ls /etc/cron.*; crontab -l

# Group memberships
id
# Check for: docker, lxd, disk, adm, sudo

# Kernel version
uname -a

# Sudo version (for CVE-2021-3156)
sudo -V | head -1
```

## Key Tools

| Tool | Purpose | Page |
|---|---|---|
| linpeas.sh | Automated privesc enumeration | [[tools/enumeration/linpeas]] |
| pspy | Unprivileged process/cron monitor | [[tools/enumeration/pspy]] |
| GTFOBins | SUID/sudo binary exploitation reference | https://gtfobins.github.io |
| logrotten | Logrotate exploit | GitHub |
| kubeletctl | Kubernetes kubelet interaction | GitHub |

## Gotchas & Notes

- Always check `sudo -l` before anything else — it often needs no password for specific commands
- Kernel exploits can crash production systems; understand the exploit before running it
- `no_root_squash` in NFS exports is a reliable escalation path if you have root on the attack host
- Password re-use across users and services is common — collect every credential found
- The `adm` group provides log read access that may reveal credentials or cron execution patterns

## Related Pages

- [[attack/linux_privesc_enumeration]]
- [[attack/linux_privesc_sudo_suid]]
- [[attack/linux_privesc_services]]
- [[attack/linux_privesc_kernel]]
- [[tools/enumeration/linpeas]]
- [[tools/enumeration/pspy]]

## Sources

- raw/linux_privilege_escalation/information_gathering_environment_enumeration.md
- raw/linux_privilege_escalation/information_gathering_linux_services_and_internals_enumeration.md
- raw/linux_privilege_escalation/information_gathering_credential_hunting.md
- raw/linux_privilege_escalation/environment_based_privalege_escalation_path_abuse.md
- raw/linux_privilege_escalation/environment_based_privalege_escalation_wildcard_abuse.md
- raw/linux_privilege_escalation/environment_based_privilege_escalation_escaping_restricted_shells.md
- raw/linux_privilege_escalation/permission_based_privilege_escalation_sudo_rights_abuse.md
- raw/linux_privilege_escalation/permission_based_privilege_escalation_privileged_groups.md
- raw/linux_privilege_escalation/permission_based_priviledge_escalation_capabilities.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_cron_job_abuse.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_docker.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_kibernetes.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_logrotate.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_lxd.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_miscellaneous_techniques.md
- raw/linux_privilege_escalation/service_based_privilege_escalation_vulnerable_services.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_kernel_exploits.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_libraries.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_shared_object_hijacking.md
- raw/linux_privilege_escalation/linux_internals_based_privilege_escalation_python_library_hijacking.md
- raw/linux_privilege_escalation/recent_0days_dirty_pipe.md
- raw/linux_privilege_escalation/recent_0days_netfilter.md
- raw/linux_privilege_escalation/recent_0days_polkit.md
- raw/linux_privilege_escalation/recent_0days_sudo.md
- raw/linux_privilege_escalation/hardening_considerations_linux_hardening.md
