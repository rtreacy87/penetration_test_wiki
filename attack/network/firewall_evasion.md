---
tags: [attack, attack/network]
module: network_enumeration_with_nmap
last_updated: 2026-05-10
source_count: 4
---

# Firewall and IDS/IPS Evasion

Techniques for bypassing firewalls, intrusion detection systems (IDS), and intrusion prevention systems (IPS) during network scanning, focused on Nmap-based methods including packet fragmentation, decoys, source port manipulation, timing control, and DNS proxying.

## Overview

Firewalls filter traffic by examining individual packets against rule sets — dropping (silently discarding) or rejecting (returning ICMP error or RST) packets that do not match allowed connections. IDS systems passively monitor traffic for known attack patterns and alert administrators. IPS systems actively block traffic matching those patterns.

The key distinction:

- **Dropped packets:** No response. Nmap retries (default 10 times), making the scan slow. Port appears `filtered`.
- **Rejected packets:** ICMP unreachable error returned immediately. Port still appears `filtered`, but the scan is fast.

Understanding which response you get helps infer the defensive posture on the target network.

---

## Identifying Firewall Rules with ACK Scans

A TCP ACK scan (`-sA`) is harder for firewalls and IDS/IPS to filter than SYN or Connect scans because it only sends packets with the `ACK` flag. Firewalls that block inbound SYN packets (new connections) often pass ACK packets because stateless firewalls cannot distinguish them from replies to established sessions.

**Result interpretation:**

- Port returns `RST` → `unfiltered` (ACK reached the host)
- No response → `filtered` (firewall is blocking ACK packets too)
- ICMP unreachable → `filtered`

```bash
# SYN scan — establishes which ports appear open/closed/filtered
sudo nmap 10.129.2.28 -p 21,22,25 -sS -Pn -n --disable-arp-ping --packet-trace

# ACK scan — maps which ports pass through the firewall at all
sudo nmap 10.129.2.28 -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```

Comparing SYN and ACK scan results reveals which ports are actively filtered vs. simply closed. A port that is `filtered` under SYN but `unfiltered` under ACK is being blocked by the firewall for inbound connections only.

---

## Decoy Scanning

Decoy scanning generates fake source IP addresses in the IP header alongside your real address, making it harder for defenders to identify which host performed the scan.

```bash
# Generate 5 random decoy IPs; real IP placed among them
sudo nmap 10.129.2.28 -p 80 -sS -Pn -n --disable-arp-ping --packet-trace -D RND:5

# Specify exact decoy IPs (useful for blending in with trusted ranges)
sudo nmap 10.129.2.28 -p 80 -sS -D 10.10.10.2,10.10.10.3,ME

# Decoys also work with ACK, ICMP, and OS detection scans
sudo nmap 10.129.2.28 -p 80 -sA -D RND:5 -Pn -n
```

**Caveats:**
- Decoy IPs must be live hosts. If they are unreachable, the target may trigger SYN-flood protection, making the real service unreachable.
- Many ISPs and routers filter spoofed packets, especially across routing boundaries. Decoys work best within the same network segment or when traffic stays inside a single AS.
- The target only responds to your real IP, so you still receive results correctly.

---

## Source IP Spoofing

When subnet-based firewall rules block your actual IP, spoofing a trusted source IP may allow access to filtered ports. This requires specifying a network interface so Nmap can send and receive traffic correctly.

```bash
# Test: port 445 appears filtered from real IP
sudo nmap 10.129.2.28 -n -Pn -p 445 -O

# Scan using a trusted subnet IP as source
sudo nmap 10.129.2.28 -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
```

| Option | Description |
|--------|-------------|
| `-S <IP>` | Spoof source IP address |
| `-e <iface>` | Send packets through specified interface |

This technique is especially useful when:
- The target has IP-based ACLs (e.g., only `10.129.2.0/24` can reach a management port)
- You have already gained a foothold inside a trusted subnet and want to test lateral access rules

---

## Source Port Manipulation (DNS Port Bypass)

Many firewall rules trust traffic originating from well-known ports, particularly DNS (`UDP/TCP 53`), which must often pass through to allow name resolution. By spoofing the source port to 53, packets may bypass rules that would otherwise block them.

```bash
# Normal scan — port appears filtered
sudo nmap 10.129.2.28 -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace

# Same scan using source port 53 — firewall trusts DNS traffic
sudo nmap 10.129.2.28 -p 50000 -sS -Pn -n --disable-arp-ping --packet-trace --source-port 53

# Confirm access by connecting directly from port 53 with ncat
ncat -nv --source-port 53 10.129.2.28 50000
```

Once you confirm that `--source-port 53` bypasses the firewall for scanning, it is likely that IDS/IPS rules for that traffic path are also less strict, making further interaction more reliable.

Other commonly trusted source ports to try:
- `80` (HTTP)
- `443` (HTTPS)
- `53` (DNS)
- `67`/`68` (DHCP)

---

## DNS Proxying

Nmap performs reverse DNS resolution by default, which leaks scan activity via DNS queries. Suppressing DNS resolution (`-n`) or routing DNS through a controlled server reduces noise and may improve accuracy in segmented networks.

```bash
# Disable DNS resolution entirely (reduces traffic and noise)
sudo nmap 10.129.2.28 -p- -n

# Use a specific DNS server (useful inside DMZ where internal DNS is more trusted)
sudo nmap 10.129.2.28 --dns-server <internal-dns-ip>

# Combine: use internal DNS + TCP port 53 as source port
sudo nmap 10.129.2.28 --dns-server <internal-dns-ip> --source-port 53 -p 445
```

In a DMZ, the company's internal DNS servers typically have broader trust in firewall rules than external resolvers. Routing Nmap's DNS queries through an internal nameserver can help reach hosts that are not visible from the external network.

---

## Fragmentation

IP fragmentation splits packets into smaller fragments that may individually pass through firewall inspection rules that look for complete packet signatures.

```bash
# Fragment packets into 8-byte fragments
sudo nmap 10.129.2.28 -f

# Fragment into 16-byte fragments (double fragmentation)
sudo nmap 10.129.2.28 -ff

# Specify maximum transmission unit
sudo nmap 10.129.2.28 --mtu 16
```

Modern stateful firewalls and IDS/IPS systems reassemble fragments before inspection, making this less effective than it was historically. However, some simpler packet filters and older appliances still do not handle fragmented packets correctly.

---

## Timing and Rate Control for IDS Evasion

Rate-based IDS/IPS systems trigger on high volumes of traffic in short windows. Slowing scans down reduces signatures that match "port sweep" patterns.

```bash
# Paranoid timing — one probe every 5 minutes (evades almost all IDS)
sudo nmap 10.129.2.28 -p- -T0

# Sneaky — one probe every 15 seconds
sudo nmap 10.129.2.28 -p- -T1

# Polite — 400ms between probes
sudo nmap 10.129.2.28 -p- -T2

# Manual rate control
sudo nmap 10.129.2.28 -p- --max-rate 10 --scan-delay 500ms
```

| Template | Alias | Delay between probes |
|----------|-------|----------------------|
| `-T0` | paranoid | 5 minutes |
| `-T1` | sneaky | 15 seconds |
| `-T2` | polite | 400ms |
| `-T3` | normal | Adaptive (default) |
| `-T4` | aggressive | Minimal |
| `-T5` | insane | Minimal (may miss ports) |

---

## Detecting IDS/IPS Presence

Unlike firewalls, IDS/IPS systems are passive monitors and cannot be directly fingerprinted with a scan. However, their presence can be inferred:

1. **Perform an aggressive scan** against a single port. If your IP becomes unreachable shortly after, an IPS blocked it.
2. **Use multiple VPS/source IPs.** If one gets blocked, rotate to another and observe whether behavior changes.
3. **Watch scan timing.** If responses suddenly stop or dramatically slow after a few packets, an IPS may be rate-limiting or blackholing your source.
4. **Compare ACK vs. SYN results.** Ports that are `filtered` under SYN but `unfiltered` under ACK suggest a stateful firewall (not just a stateless filter).

Once IPS presence is confirmed, all further scanning must be done quietly — lower timing templates, decoys, source port manipulation, and manual interaction with `nc`/`ncat` instead of automated sweeps.

---

## Gotchas & Notes

- **Decoys require live hosts.** Nmap does not verify decoy IPs are live; using dead IPs can trigger SYN-flood defenses on the target.
- **Source port spoofing is not the same as IP spoofing.** `--source-port` only changes the TCP/UDP source port, not the IP source address. Responses still return to your actual IP.
- **Fragmentation is largely defeated by modern firewalls.** Use it as a last resort or when targeting legacy infrastructure.
- **Low timing templates slow investigations significantly.** `-T0` for 65535 ports would take weeks. Use only for targeted probes.
- **ACK scans do not confirm open ports.** They only confirm whether a port passes through the firewall. Use a SYN or Connect scan for open port verification.
- **IPS detection by getting blocked.** If you suddenly lose access to the target network, your IP has likely been blocked by an IPS. Rotate source IPs and continue more quietly.
- **DNS source port bypass effectiveness varies.** Not all firewalls trust source port 53 universally; effectiveness depends on the specific rule set.

---

## Related Pages

- [[tools/enumeration/nmap]]
- [[tools/netcat]]
- [[enumeration/dns]]

## Sources

- raw/network_enumeration_with_nmap/bypass_security_measures_firewall_evasion.md
- raw/network_enumeration_with_nmap/host_enumeration_host_and_port_scanning.md
- raw/network_enumeration_with_nmap/bypass_security_measures_easy_lab.md
- raw/network_enumeration_with_nmap/bypass_security_measures_medium_lab.md
