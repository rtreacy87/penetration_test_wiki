---
tags: [tool]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# smbclient

smbclient is a command-line SMB client for listing shares, browsing directories, downloading/uploading files, and interacting with SMB servers from Linux.

## Overview

Part of the Samba suite. Allows connecting to SMB shares using null sessions (anonymous) or with credentials. Functions similarly to an FTP client once connected.

## Commands / Syntax

```bash
# List all shares (null session)
smbclient -N -L //10.129.14.128

# List all shares (with credentials)
smbclient -L //10.129.14.128 -U username

# Connect to a specific share (null session)
smbclient -N //10.129.14.128/notes

# Connect with credentials
smbclient //10.129.14.128/notes -U username

# Connect with domain
smbclient //10.129.14.128/C$ -U DOMAIN/username
```

### Inside an smbclient Session

```
smb: \> help              # List all available commands
smb: \> ls                # List files in current directory
smb: \> cd <dir>          # Change directory
smb: \> get <file>        # Download file
smb: \> put <file>        # Upload file
smb: \> mget <pattern>    # Download multiple files
smb: \> mput <pattern>    # Upload multiple files
smb: \> mkdir <dir>       # Create directory
smb: \> del <file>        # Delete file
smb: \> recurse           # Toggle recursive mode
smb: \> prompt            # Toggle prompt for mget/mput
smb: \> !ls               # Run local shell command (! prefix)
smb: \> !cat <file>       # Print local file
smb: \> quit              # Exit
```

## Flags & Options

| Flag | Description |
|------|-------------|
| `-N` | No password (null session) |
| `-L //host` | List shares |
| `-U <user>` | Username |
| `-W <workgroup>` | Workgroup/domain |
| `-p <port>` | Non-standard port (default 445) |
| `--password <pass>` | Specify password |
| `-k` | Use Kerberos authentication |
| `-c <command>` | Execute single command non-interactively |

## Gotchas & Notes

- `!<command>` prefix executes commands on the **local** machine, not the remote — useful for checking what was downloaded without leaving the session.
- Recursive download: enable `recurse` and `prompt` then use `mget *` to download all files in the current share directory tree.
- `SMB1 disabled` message in list output is normal — smbclient defaults to SMB2+.
- Null session (`-N`) without `-L` will prompt for a password; always combine with `-L` or supply credentials.

## Related Pages

- [[enumeration/smb]]
- [[tools/enum4linux]]
- [[tools/rpcclient]]
- [[tools/netexec]]

## Sources

- raw/footprinting/host_based_enumeration_smb.md
