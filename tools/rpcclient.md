---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# rpcclient

rpcclient is a Samba tool for executing MS-RPC functions against SMB servers, enabling user, group, share, and domain enumeration via null or authenticated sessions.

## Overview

rpcclient connects to the IPC$ share and issues MSRPC calls directly. It is particularly useful for user enumeration via RID brute-forcing when `enumdomusers` returns limited results.

## Commands / Syntax

```bash
# Null session connection
rpcclient -U "" 10.129.14.128
# Press Enter at password prompt

# With credentials
rpcclient -U "username%password" 10.129.14.128

# Single command (non-interactive)
rpcclient -N -U "" 10.129.14.128 -c "enumdomusers"

# RID brute-force loop (finds users not returned by enumdomusers)
for i in $(seq 500 1100); do
    rpcclient -N -U "" 10.129.14.128 \
    -c "queryuser 0x$(printf '%x\n' $i)" \
    | grep "User Name\|user_rid\|group_rid" && echo ""
done
```

### RPC Commands (Inside Session)

```
rpcclient$> srvinfo                         # Server info (platform, OS version, type)
rpcclient$> enumdomains                     # List domains
rpcclient$> querydominfo                    # Domain, server, user count
rpcclient$> netshareenumall                 # All shares with paths
rpcclient$> netsharegetinfo <share>         # Details for specific share (ACLs, path)
rpcclient$> enumdomusers                    # All domain users (may be incomplete)
rpcclient$> queryuser <RID>                 # User details by hex RID (e.g., 0x3e9)
rpcclient$> queryuser 0x3e8                 # RID 1000 — first non-default local user
rpcclient$> enumdomgroups                   # Domain groups
rpcclient$> querygroup <RID>                # Group details by RID
rpcclient$> enumalsgroups domain            # Local groups
rpcclient$> enumalsgroups builtin           # Builtin groups
rpcclient$> lookupnames <username>          # Get SID for username
rpcclient$> lookupsids <SID>                # Get username for SID
rpcclient$> getdompwinfo                    # Password policy
rpcclient$> lsaenumsid                      # Enumerate SIDs
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-U <user>` | Username (use `""` for null session) |
| `-N` | No password |
| `--password <pass>` | Supply password |
| `-c "<command>"` | Execute single command and exit |
| `-W <workgroup>` | Workgroup/domain |
| `-k` | Use Kerberos |

## Gotchas & Notes

- **RID 0x3e8 = RID 1000**: The first non-built-in local user on Linux/Samba. Built-in accounts start below 0x1f5 (501).
- **RID brute-force range**: 500–1100 covers most environments. Increase the upper bound on large domains with many users.
- `queryuser <RID>` returns full details: username, home directory, profile path, logon/logoff times, password last set, bad password count.
- `netshareenumall` shows all shares including hidden ones (ending in `$`) and their UNC paths — more complete than smbclient `-L`.
- `netsharegetinfo <share>` returns the ACL for the share (DACL with SID and permission bits).
- **Not all commands require admin**: `queryuser`, `netshareenumall`, and `enumdomusers` typically work with null sessions on misconfigured Samba servers.
- `srvinfo` reveals OS version — useful for identifying outdated systems.

## Related Pages

- [[enumeration/smb]]
- [[tools/smbclient]]
- [[tools/enum4linux]]
- [[tools/crackmapexec]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
