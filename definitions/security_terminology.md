---
tags: [definition, concept, reference]
module: core
last_updated: 2026-05-11
source_count: 0
---

# Security Terminology

Pentest-relevant security terms, vulnerability classes, and scoring systems — what they mean and why they matter in an engagement.

## Vulnerability Identifiers and Scoring

### CVE (Common Vulnerabilities and Exposures)

A unique identifier assigned to a publicly disclosed vulnerability. Format: `CVE-YYYY-NNNNN`. Issued by CVE Numbering Authorities (CNAs) and maintained by MITRE.

**Pentester use:** CVE numbers are the search key for PoCs, Metasploit modules, and exploit-db entries. Always search by CVE on exploit-db.com, GitHub, and `searchsploit`.

### CVSS (Common Vulnerability Scoring System)

A 0–10 numerical score describing a vulnerability's severity. CVSS v3 produces a Base Score from: Attack Vector, Attack Complexity, Privileges Required, User Interaction, Scope, Confidentiality/Integrity/Availability impact.

| Score | Severity |
|-------|----------|
| 0.0 | None |
| 0.1–3.9 | Low |
| 4.0–6.9 | Medium |
| 7.0–8.9 | High |
| 9.0–10.0 | Critical |

**Pentester use:** CVSS ≥ 9.0 with AV:N (network-accessible) and PR:N (no privileges required) is your priority target list.

### CWE (Common Weakness Enumeration)

Classifies the *type* of weakness (e.g., CWE-89: SQL Injection, CWE-79: XSS, CWE-22: Path Traversal). Different from CVE (specific instance) — CWE is the category.

### OWASP Top 10

Annual list of the most critical web application security risks. Current categories include:
- A01: Broken Access Control
- A02: Cryptographic Failures
- A03: Injection (SQLi, XSS, command injection)
- A04: Insecure Design
- A05: Security Misconfiguration
- A06: Vulnerable and Outdated Components
- A07: Identification and Authentication Failures
- A08: Software and Data Integrity Failures
- A09: Security Logging and Monitoring Failures
- A10: Server-Side Request Forgery (SSRF)

---

## Execution Vulnerability Classes

### RCE (Remote Code Execution)

Arbitrary OS command or code execution on the target from a remote position, without physical access. The highest-impact individual vulnerability class.

**Examples:** Apache Log4Shell, EternalBlue (MS17-010), OpenSMTPD CVE-2020-7247, MSSQL xp_cmdshell when writable.

### LFI (Local File Inclusion)

A web vulnerability where user-controlled input is passed to a file inclusion function, allowing the attacker to read arbitrary local files (e.g., `/etc/passwd`, SSH keys, web app configs).

**Escalation path:** LFI + writable log file (or `/proc/self/environ`) → RCE via log poisoning.

**Example:** `https://target.com/page.php?file=../../../../etc/passwd`

### RFI (Remote File Inclusion)

Like LFI but includes a file from a remote URL. Requires `allow_url_include = On` in PHP (disabled by default since PHP 5.2). Near-instant RCE if exploitable.

### Directory Traversal (Path Traversal)

User-supplied input contains `../` sequences that escape the intended directory. Allows reading or writing files outside the web root.

**Example:** `GET /download?file=../../Windows/win.ini`

### SSRF (Server-Side Request Forgery)

The server is tricked into making HTTP requests to arbitrary destinations, including internal services (169.254.169.254 metadata, internal APIs, localhost). The server's requests may bypass firewall rules that block external access.

**Escalation:** SSRF to cloud metadata → IAM credentials → full cloud account compromise.

### IDOR (Insecure Direct Object Reference)

Access control flaw where a user can reference an object they shouldn't have access to by manipulating an identifier (e.g., changing `?user_id=1234` to `?user_id=1235`). Purely an authorization issue.

---

## Injection Classes

### SQLi (SQL Injection)

User input is inserted into an SQL query without sanitization. Types: Classic (UNION-based), Error-based, Blind Boolean, Time-based Blind, Out-of-band (DNS/HTTP exfiltration).

**Pentester tool:** `sqlmap` for automated detection and exploitation.

### XSS (Cross-Site Scripting)

Attacker-controlled JavaScript executes in a victim's browser. Types: Reflected (URL parameter), Stored (database), DOM-based (client-side JS manipulation).

**Impact:** Session hijacking, credential harvesting, CSRF bypass, keylogging.

### XXE (XML External Entity)

When an XML parser processes external entity references, an attacker can read local files (`file:///etc/passwd`) or trigger SSRF via crafted XML.

### SSTI (Server-Side Template Injection)

User input is rendered inside a server-side template engine (Jinja2, Twig, Freemarker, etc.), allowing code execution in the template context — often escalating to RCE.

**Detection probe:** `{{7*7}}` or `${7*7}` in any user-controlled field that appears reflected.

### Command Injection

User input is passed to a shell command via `system()`, `exec()`, `popen()`, etc. without sanitization. Often achieved by appending `;`, `|`, `&&`, or backticks.

**Example:** `ping -c 1 10.10.10.1; id`

---

## Post-Exploitation Terms

### Privilege Escalation (PrivEsc)

Gaining higher privileges than initially obtained — typically from an unprivileged shell to root (Linux) or SYSTEM/Administrator (Windows).

**Categories:** Kernel exploits, SUID/sudo misconfigs, credential reuse, service exploits, writeable paths.

### Lateral Movement

Moving from one compromised host to other hosts in the same network, typically using captured credentials or hashes. See [[attack/smb]] for SMB-based lateral movement.

### Persistence

Maintaining access after the initial foothold, surviving reboots and session timeouts. Methods: cron jobs, startup scripts, registry run keys, scheduled tasks, SSH authorized_keys, web shells.

### C2 (Command and Control)

Infrastructure used to maintain communication with compromised hosts. Implants "phone home" to the C2 to receive commands and exfiltrate data. Examples: Cobalt Strike, Metasploit listener, Sliver, Havoc.

### Pivoting

Using a compromised host as a relay to reach otherwise inaccessible network segments. Tools: `ssh -L`/`-R`/`-D`, `chisel`, `ligolo-ng`, Metasploit's `route add`.

### Pass-the-Hash (PTH)

Using a captured NT hash (without cracking it) to authenticate via NTLM directly. See [[definitions/auth_protocols]] and [[attack/smb]].

### Kerberoasting

Requesting Kerberos TGS tickets for service principal names (SPNs) and cracking them offline. The tickets are encrypted with the service account's NT hash. See [[definitions/auth_protocols]].

---

## Defense Terms (Know What You're Up Against)

| Term | What it does | Bypass notes |
|------|-------------|--------------|
| **IDS** (Intrusion Detection System) | Monitors and alerts on suspicious traffic | Fragmentation, encoding, timing, protocol anomalies |
| **IPS** (Intrusion Prevention System) | Actively blocks detected attacks | Same as IDS evasion; also protocol tunneling |
| **WAF** (Web Application Firewall) | Filters HTTP/S based on rules | Encoding, case variation, chunked requests, H2C smuggling |
| **EDR** (Endpoint Detection & Response) | Behavioral monitoring on endpoints | AMSI bypass, process injection, living-off-the-land |
| **AMSI** (Antimalware Scan Interface) | Windows hook for AV to inspect script content | Patching AMSI in memory, obfuscation, reflection |
| **AppLocker / WDAC** | Whitelist execution control on Windows | Trusted binary abuse (LOLBAS), DLL sideloading |
| **NLA** (Network Level Authentication) | Pre-login auth for RDP | Requires valid creds before showing the login screen |

## Related pages

- [[definitions/network_protocols]]
- [[definitions/tcp_flags]]
- [[definitions/auth_protocols]]
- [[attack/_overview]]
- [[attack/smb]]
- [[attack/sql_databases]]
