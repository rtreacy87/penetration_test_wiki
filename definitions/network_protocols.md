---
tags: [definition, concept, reference]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Network Protocols

Layer 3 and Layer 4 protocols — what they are, how they behave, and why a pentester cares about each.

## IP (Internet Protocol)

The Layer 3 addressing and routing protocol. All traffic rides on IP.

**Pentester relevance:** Source IP can be spoofed at L3 for scanning deception (`nmap -D` decoy scans). ICMP (below) is technically a sub-protocol of IP. IP fragmentation is used to evade IDS/firewall inspection.

## TCP (Transmission Control Protocol)

Connection-oriented, reliable transport. Establishes a three-way handshake (SYN → SYN-ACK → ACK) before data flows.

**Pentester relevance:** Most attack surface runs over TCP. The handshake creates state — firewalls track it. Half-open SYN scans (`nmap -sS`) send SYN but never complete the handshake, making them less noisy than full connects. See [[definitions/tcp_flags]] for flag-level detail.

Key ports: virtually all application-layer services (see [[ports/common_ports]]).

## UDP (User Datagram Protocol)

Connectionless, no handshake, no delivery guarantee. Faster but unreliable.

**Pentester relevance:** Many infrastructure services run over UDP: DNS (53), SNMP (161/162), DHCP (67/68), TFTP (69), NTP (123), IPMI (623). UDP is harder to scan — no open/closed confirmation means Nmap falls back on ICMP port-unreachable responses, which firewalls often drop. Use `nmap -sU` but expect slow results. SNMP community string attacks and DNS amplification both exploit UDP services.

## ICMP (Internet Control Message Protocol)

Used for diagnostics and error reporting. Not a transport protocol — carries no port numbers.

**Pentester relevance:** 
- `ping` (ICMP Echo Request/Reply) — host discovery
- ICMP `type 3 code 3` (Port Unreachable) — signals a UDP port is closed
- ICMP `type 11` (Time Exceeded) — used by `traceroute`
- Many firewalls block inbound ICMP; `-Pn` in Nmap skips the ping probe

Key ICMP types:

| Type | Name | Pentester use |
|------|------|---------------|
| 0 | Echo Reply | Confirms host is up |
| 3 | Destination Unreachable | Signals closed UDP port (code 3) |
| 8 | Echo Request | Host discovery (ping) |
| 11 | Time Exceeded | Traceroute hop enumeration |

## SCTP (Stream Control Transmission Protocol)

Multi-stream, connection-oriented transport designed for telecom signaling (SS7). Less common than TCP/UDP.

**Pentester relevance:** Found in VoIP, telecom, and some cloud infrastructure. Nmap supports SCTP scanning (`-sY` for INIT scan, `-sZ` for COOKIE-ECHO). SCTP lacks the mature filtering that TCP has — misconfigured SCTP endpoints can bypass firewall rules written only for TCP/UDP. Rare but high-value when encountered.

## ARP (Address Resolution Protocol)

Maps IP addresses to MAC addresses on a local segment (L2).

**Pentester relevance:** ARP has no authentication — ARP spoofing/poisoning lets an on-path attacker redirect Layer-2 traffic. Responder uses ARP implicitly; `ettercap -M arp` is an explicit ARP poisoning attack. Also used for host discovery on local subnets (`arp-scan -l`, `nmap -PR`).

## QUIC

Google-originated protocol: UDP-based, multiplexed, encrypted (TLS 1.3 built in). HTTP/3 runs over QUIC.

**Pentester relevance:** QUIC traffic on UDP/443 can bypass traditional HTTP inspection. Some WAFs and proxies don't yet handle QUIC. If a target has HTTP/3 enabled, QUIC scanning may reveal services that TCP-only scans miss. Still emerging in penetration testing tooling.

## GRE / IPSec / L2TP (Tunneling protocols)

Encapsulate other protocols for VPN or point-to-point tunneling.

**Pentester relevance:** VPN infrastructure is often less hardened than perimeter firewalls. GRE (IP proto 47) and ESP (IP proto 50) are worth probing during recon. IPSec IKE on UDP/500 and UDP/4500 can be targeted with ike-scan for aggressive-mode pre-shared key cracking.

## Related pages

- [[definitions/tcp_flags]]
- [[definitions/auth_protocols]]
- [[ports/common_ports]]
- [[tools/enumeration/nmap]]
- [[enumeration/_overview]]
