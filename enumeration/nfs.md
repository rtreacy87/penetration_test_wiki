---
tags: [enumeration, enumeration/nfs]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# NFS Enumeration

NFS enumeration: showmount, mounting shares, UID/GID-based access, and privilege escalation via uid spoofing.

## Overview

NFS (Network File System) allows file systems to be shared over a network as if they were local. It is a Linux/Unix protocol (SMB is the Windows equivalent). Key ports: **TCP/UDP 111** (RPC portmapper) and **TCP/UDP 2049** (NFS).

NFS uses UID/GID-based authorization — there is no separate authentication mechanism. If you can create a local user with the same UID as the NFS share owner, you can access that user's files. This is the primary privilege escalation vector.

## Key Concepts / Techniques

### NFS Versions

| Version | Key Features |
|---------|-------------|
| NFSv2 | Legacy; UDP only |
| NFSv3 | Variable file size; better error reporting; not fully v2-compatible |
| NFSv4 | Kerberos support; stateful; works through firewalls; single port 2049; ACLs |

NFSv4.1 adds pNFS (parallel NFS) for clustered deployments.

### Authentication Model

NFSv2 and NFSv3 authenticate the **client computer**, not the user. NFSv4 adds user authentication (Kerberos-optional). Without Kerberos, access is controlled entirely by UID/GID.

**Critical**: If you have root on your attack box, you can `useradd` with a specific UID to match the owner of files on the NFS share and access them.

### `/etc/exports` Options

| Option | Description |
|--------|-------------|
| `rw` | Read and write access |
| `ro` | Read-only access |
| `sync` | Synchronous writes (slower, safer) |
| `async` | Asynchronous writes (faster, risky) |
| `no_subtree_check` | Disable subdirectory tree checking |
| `root_squash` | Map root UID (0) to anonymous UID — prevents root access to NFS files |
| `no_root_squash` | Root retains UID 0 on NFS shares — dangerous |
| `insecure` | Allow connections from ports > 1024 |
| `nohide` | Export sub-mounted filesystems |

### SUID Exploitation via NFS

If you have SSH access and need to read files owned by another user:
1. Mount the NFS share on your attack machine
2. Create a local user with the target UID
3. Place a SUID shell binary on the NFS share owned by that UID
4. Execute via the SSH session

## Commands / Syntax

```bash
# Scan NFS ports
sudo nmap 10.129.14.128 -p111,2049 -sV -sC

# Enumerate NFS with NSE scripts
sudo nmap --script nfs* 10.129.14.128 -sV -p111,2049

# Show available NFS exports
showmount -e 10.129.14.128

# Mount an NFS share
mkdir target-NFS
sudo mount -t nfs 10.129.14.128:/ ./target-NFS/ -o nolock
cd target-NFS

# List files with usernames
ls -l mnt/nfs/

# List files with numeric UIDs/GIDs (to identify spoofing targets)
ls -n mnt/nfs/

# Unmount the share
cd ..
sudo umount ./target-NFS
```

## Flags & Options

### Nmap NFS NSE Scripts

| Script | Description |
|--------|-------------|
| `nfs-ls` | List files on NFS exports with permissions |
| `nfs-showmount` | Equivalent to showmount -e |
| `nfs-statfs` | Filesystem statistics (size, used, available) |
| `rpcinfo` | List all running RPC services and their ports |

### mount

| Flag | Description |
|------|-------------|
| `-t nfs` | Specify NFS type |
| `-o nolock` | Disable locking (required for some shares) |
| `-o vers=3` | Force NFSv3 |
| `-o nfsvers=4` | Force NFSv4 |

## Version Detection & Exploit Research

NFS version information is exposed via the RPC portmapper on port 111 — `rpcinfo` reveals which NFS versions (2, 3, 4) are registered and active without authentication. The kernel NFS daemon (`nfsd`) version can also be obtained via nmap. NFS CVEs are typically tied to kernel versions rather than a standalone NFS package, so identifying the kernel version through other means (SSH banner, SNMP) is important context.

### Extracting Version Information

| Method | Command | What It Reveals |
|--------|---------|-----------------|
| Nmap RPC scan | `nmap -p111,2049 -sV -sC <IP>` | NFS version(s) in use, RPC program numbers |
| rpcinfo | `rpcinfo -p <IP>` | All registered RPC services + versions |
| NSE rpcinfo | `nmap -p111 --script rpcinfo <IP>` | NFS version list, additional RPC services |
| NSE nfs-ls | `nmap --script nfs-ls <IP>` | Export list + access flags |
| Nmap NSE full | `nmap --script nfs* -p111,2049 <IP>` | Version, exports, filesystem stats |

**What to record:**
- NFS versions active (v2 = legacy/insecure, v3 = common, v4 = modern)
- Kernel-level NFS daemon version if exposed in nmap output
- `no_root_squash` in export flags — highest-priority finding

### Searching for Exploits

```bash
# Searchsploit
searchsploit nfs
searchsploit nfsd
searchsploit "network file system"

# Metasploit
msf6> search type:exploit name:nfs
```

### Notable CVEs

| CVE | Affected | Impact |
|-----|----------|--------|
| CVE-2017-7895 | Linux kernel < 4.11 | NFSv2/v3 `nfsd` integer overflow — remote kernel exploit |
| CVE-2019-3010 | Solaris 11.x | NFS privilege escalation via symlink race |
| CVE-2021-3178 | Linux kernel < 5.10.9 | NFS client path traversal — read out-of-bounds |
| CVE-2022-24448 | Linux kernel < 5.16.5 | NFSv4 open() flaw — local privilege escalation |
| `no_root_squash` (misconfiguration) | Any NFS export | UID 0 retained — equivalent to local root on the share |

## Gotchas & Notes

- **`root_squash` prevents root escalation** on the NFS share. If this option is absent (`no_root_squash`), mounting the share as root on your local machine gives you root-level access to all share contents.
- **SSH key exposure**: NFS shares frequently contain `.ssh/` directories with private keys. If `id_rsa` is world-readable on a misconfigured NFS share, you can log into the server without a password.
- **UID spoofing**: When `ls -n` shows UID 1000 owns a file, `useradd -u 1000 attackuser` on your local machine then gives you that user's access to those files.
- **backup.sh with root_squash**: If root_squash is enabled, even as root on your local machine, you cannot edit files owned by UID 0 on the share. Plan around this.
- **SUID shell trick**: Upload a bash shell binary to the share (`cp /bin/bash ./bash; chmod u+s ./bash`), then execute it via SSH to gain the share owner's privileges.
- NFSv4 uses only port 2049 — easier to firewall, meaning you are more likely to encounter NFSv4 on internet-exposed systems.

## Related Pages

- [[enumeration/_overview]]
- [[enumeration/smb]]

## Sources

- raw/footprinting/host_based_enumeration_nfs.md
