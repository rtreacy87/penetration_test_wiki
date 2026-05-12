---
tags: [reference, enumeration, definition]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Common Ports

Master reference table for ports a penetration tester will encounter, organized by service category.

## Remote Access

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 22 | TCP | SSH | Default creds, key auth bypass, tunneling/SOCKS proxy, user enum (pre-auth timing) |
| 23 | TCP | Telnet | Cleartext credentials — sniff or brute force; often left open on network gear |
| 3389 | TCP | RDP | Password spray (Crowbar/Hydra), PTH (xfreerdp), session hijacking, BlueKeep CVE-2019-0708 |
| 5985 | TCP | WinRM (HTTP) | evil-winrm with creds or hash; domain lateral movement |
| 5986 | TCP | WinRM (HTTPS) | Same as 5985 over TLS |
| 5900 | TCP | VNC | Default/weak password; no auth configs; screen scraping |
| 2222 | TCP | SSH (alt) | Non-standard SSH; same attacks as 22 |
| 8291 | TCP | Mikrotik Winbox | RouterOS management; CVE-2018-14847 credential disclosure |

## File Transfer

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 20 | TCP | FTP data | Active mode data channel |
| 21 | TCP | FTP control | Anonymous login, brute force (Medusa), bounce attack, CVE-2022-22836 |
| 69 | UDP | TFTP | No authentication; read/write arbitrary files if writable |
| 445 | TCP | SMB | Null sessions, spray, RCE (psexec), PTH, LLMNR/relay, EternalBlue MS17-010 |
| 139 | TCP | NetBIOS/SMB | Legacy SMB (SMBv1); same attack surface as 445 |
| 2049 | TCP/UDP | NFS | Mount remote shares; UID/GID impersonation for file access |
| 548 | TCP | AFP (Apple Filing Protocol) | Older macOS file shares; default/weak creds |

## Web Services

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 80 | TCP | HTTP | Full web attack surface: SQLi, XSS, LFI, SSRF, directory brute-force |
| 443 | TCP | HTTPS | Same as 80 + cert information disclosure (domains via SNI/SAN) |
| 8080 | TCP | HTTP alt | Dev/proxy ports; often less hardened than prod |
| 8443 | TCP | HTTPS alt | Same as 8080 over TLS; common for admin panels |
| 8888 | TCP | HTTP alt | Jupyter Notebook, various dev servers |
| 3000 | TCP | HTTP (Node/Grafana) | Grafana, Node.js apps, dev environments |
| 8008 | TCP | HTTP alt | Common on embedded devices, cameras |
| 9200 | TCP | Elasticsearch | Unauthenticated API common; full database read/write |
| 9090 | TCP | Prometheus/Cockpit | Metrics endpoints; info disclosure; Cockpit Linux web console |

## Email

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 25 | TCP | SMTP | User enumeration (VRFY/RCPT), open relay, CVE-2020-7247 OpenSMTPD RCE |
| 465 | TCP | SMTPS | Encrypted SMTP; same attacks over TLS |
| 587 | TCP | SMTP submission | Auth relay; password spray with Hydra/o365spray |
| 110 | TCP | POP3 | Read mailbox after cred compromise; Hydra spray |
| 995 | TCP | POP3S | Encrypted POP3 |
| 143 | TCP | IMAP | Full mailbox access; openssl or curl interaction |
| 993 | TCP | IMAPS | Encrypted IMAP |

## Database

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 1433 | TCP | MSSQL | xp_cmdshell RCE, xp_dirtree hash theft, IMPERSONATE privesc, Windows auth |
| 2433 | TCP | MSSQL (hidden) | "Hidden" MSSQL instance mode |
| 3306 | TCP | MySQL | SELECT INTO OUTFILE webshell, LOAD_FILE read, brute force |
| 5432 | TCP | PostgreSQL | COPY TO/FROM PROGRAM RCE (if superuser), brute force |
| 1521 | TCP | Oracle TNS | SID brute-force, ODAT exploitation, sysdba escalation |
| 27017 | TCP | MongoDB | Unauthenticated access common; full DB dump |
| 6379 | TCP | Redis | No-auth common; RCE via config set + cron/authorized_keys write |
| 5984 | TCP | CouchDB | HTTP API; admin party (no-auth) misconfigs |

## Directory Services / Identity

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 88 | TCP/UDP | Kerberos | Kerberoasting, AS-REP roasting, Pass-the-Ticket, Golden/Silver Ticket |
| 389 | TCP/UDP | LDAP | Null bind enumeration, ldapdomaindump, credential stuffing |
| 636 | TCP | LDAPS | Encrypted LDAP; same enumeration over TLS |
| 3268 | TCP | LDAP Global Catalog | Domain-wide AD queries |
| 3269 | TCP | LDAPS Global Catalog | Encrypted global catalog |
| 464 | TCP/UDP | kpasswd (Kerberos) | Password change; exploitable in some scenarios |

## Infrastructure / Network Services

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 53 | TCP/UDP | DNS | Zone transfer (AXFR), subdomain brute-force, subdomain takeover |
| 67/68 | UDP | DHCP | Rogue DHCP server for MITM; info disclosure of network topology |
| 123 | UDP | NTP | Info disclosure (NTP monlist); amplification in DoS |
| 161 | UDP | SNMP | Community string brute-force; MIB walk for host info |
| 162 | UDP | SNMP trap | Trap receiver; listen for management events |
| 500 | UDP | IKE (IPSec) | Aggressive mode PSK crack with ike-scan + hashcat |
| 4500 | UDP | IKE NAT-T | IPSec over NAT |
| 179 | TCP | BGP | Route injection if peering session accessible |

## Management / Out-of-Band

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 623 | UDP | IPMI | RAKP authentication → hash dump → crack; default creds (ADMIN/ADMIN) |
| 8080 | TCP | HTTP proxy / Tomcat | Manager app brute force → WAR file deploy → RCE |
| 4848 | TCP | GlassFish admin | Default admin:adminadmin; deploy WAR for RCE |
| 8161 | TCP | ActiveMQ admin | Default admin:admin; exploit path to RCE |
| 9000 | TCP | PHP-FPM / SonarQube | FPM: Fastcgi exploit (Gopherus); SonarQube: cred brute |
| 2375 | TCP | Docker (unauth) | Unauthenticated Docker daemon → container escape → host root |
| 2376 | TCP | Docker (TLS) | Same as 2375 over TLS; check cert requirements |

## Windows-Specific

| Port | Protocol | Service | Pentester Interest |
|------|----------|---------|-------------------|
| 135 | TCP | MSRPC / RPC Endpoint Mapper | Enumerates RPC services; used by impacket wmiexec |
| 137/138 | UDP | NetBIOS Name/Datagram | Name resolution poisoning (Responder) |
| 139 | TCP | NetBIOS Session | Legacy SMB; same attacks as 445 |
| 445 | TCP | SMB over TCP | (See File Transfer above) |
| 593 | TCP | HTTP RPC | DCOM over HTTP |
| 1801 | TCP | MSMQ | Message queue; CVE-2023-21554 (QueueJumper) |

## Quick Nmap Port Lists

```bash
# Top 100 ports (fast initial scan)
nmap -F 10.129.14.128

# All common service ports
nmap -p 21,22,23,25,53,80,88,110,111,135,137,139,143,161,389,443,445,465,587,623,636,993,995,1433,1521,2049,2375,3000,3268,3306,3389,5432,5900,5985,6379,8080,8443,9200 10.129.14.128

# Full TCP scan (slow but complete)
nmap -p- 10.129.14.128

# UDP top 20
nmap -sU --top-ports 20 10.129.14.128
```

## Related pages

- [[definitions/network_protocols]]
- [[definitions/tcp_flags]]
- [[enumeration/_overview]]
- [[tools/enumeration/nmap]]
- [[attack/_overview]]
