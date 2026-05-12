---
tags: [definition, concept, reference]
module: core
last_updated: 2026-05-11
source_count: 0
---

# TCP Flags

The 8 bits in the TCP control field, what they mean, and how pentesters use (and abuse) them.

## The flags

| Flag | Full name | Bit | Purpose |
|------|-----------|-----|---------|
| SYN | Synchronize | 0x02 | Initiate a connection; carries initial sequence number |
| ACK | Acknowledge | 0x10 | Acknowledge received data; set on every packet after the handshake |
| FIN | Finish | 0x01 | Gracefully close connection |
| RST | Reset | 0x04 | Abort connection immediately; no grace period |
| PSH | Push | 0x08 | Tell receiver to pass data to application immediately (don't buffer) |
| URG | Urgent | 0x20 | Urgent pointer field is significant; rarely used |
| ECE | ECN-Echo | 0x40 | Explicit Congestion Notification signaling |
| CWR | Congestion Window Reduced | 0x80 | Sender reduced congestion window |

## Three-way handshake

```
Client  →  SYN          →  Server   (seq=x)
Client  ←  SYN-ACK      ←  Server   (seq=y, ack=x+1)
Client  →  ACK          →  Server   (ack=y+1)
        [ data flows ]
```

Understanding the handshake is foundational for port scanning — different responses to SYN reveal the port state.

## Port state inference from flag responses

| Scan type | Probe sent | Open response | Closed response | Filtered |
|-----------|-----------|---------------|-----------------|----------|
| SYN scan (`-sS`) | SYN | SYN-ACK | RST | No response / ICMP unreachable |
| Connect scan (`-sT`) | SYN | SYN-ACK → completes ACK | RST | Timeout |
| ACK scan (`-sA`) | ACK | RST (unfiltered) | RST (unfiltered) | No response (filtered) |
| FIN scan (`-sF`) | FIN | No response (open\|filtered) | RST | No response |
| NULL scan (`-sN`) | No flags | No response (open\|filtered) | RST | No response |
| XMAS scan (`-sX`) | FIN+PSH+URG | No response (open\|filtered) | RST | No response |
| Window scan (`-sW`) | ACK | RST with non-zero window = open | RST with zero window = closed | No response |

## Scan type selection for pentesters

**SYN scan (`-sS`)** — default when run as root. Never completes the handshake, so many logging systems don't record a full connection. Reliable and fast.

**Connect scan (`-sT`)** — used when you don't have raw socket privileges (non-root). Completes the full handshake — noisier but more accurate through some proxies.

**ACK scan (`-sA`)** — used to map firewall rules, not to find open ports. Determines which ports are *filtered* vs *unfiltered* (whether the firewall blocks the packet).

**FIN / NULL / XMAS** — stealth scans that exploit a quirk in RFC 793: compliant TCP stacks don't respond to these when a port is open, but send RST when closed. Doesn't work against Windows (Windows always responds with RST regardless).

**Idle scan (`-sI`)** — routes SYN packets through a zombie host to mask the scanner's IP. Requires a suitable idle host with predictable IP ID sequence.

## RST vs FIN: connection termination

- **FIN** = polite close. Both sides exchange FIN/ACK pairs (four-way teardown). Application has time to flush buffers.
- **RST** = hard abort. Session is dropped immediately. Useful for pentesters to recognise: an RST to a connection attempt typically means "port closed but host is up."

## PSH and URG in practice

**PSH** is routinely set on the last segment of an application message to flush the receive buffer immediately. Pentesters rarely need to manipulate it directly.

**URG** is almost never used in modern protocols. Its presence in unexpected traffic can indicate tunneling or protocol anomaly.

## Related pages

- [[definitions/network_protocols]]
- [[tools/enumeration/nmap]]
- [[attack/network/firewall_evasion]]
- [[ports/common_ports]]
