---
tags: [tool, enumeration]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 5
---

# Nessus

Tenable's commercial vulnerability scanner. Nessus Essentials (free) scans up to 16 hosts; Nessus Professional/Expert remove the limit. Industry-standard for authenticated credentialed scanning, compliance audits, and CVE-based vulnerability enumeration.

> **GUI-heavy tool.** Nessus's primary interface is a web app at `https://localhost:8834`. For headless/automated workflows see the [CLI Alternatives](#cli-alternatives) section below.

## Installation

```bash
# Check if installed
which nessus || systemctl status nessusd

# Download the Debian package from https://www.tenable.com/downloads/nessus
# Then install:
sudo dpkg -i Nessus-<version>-debian10_amd64.deb

# Start the service
sudo systemctl start nessusd
sudo systemctl enable nessusd

# Access the UI
# https://localhost:8834
```

**Activation code:** Request a free Nessus Essentials code at `https://www.tenable.com/products/nessus/nessus-essentials`. Enter it on first launch. Plugins will compile (~15 min) before the UI is usable.

---

## Scan Templates

Nessus organizes templates into three categories:

| Category | Templates |
|----------|-----------|
| **Discovery** | Host Discovery — alive hosts only, no vulnerability checks |
| **Vulnerabilities** | Basic Network Scan, Advanced Scan, Malware Scan, Web Application Tests, CVE-specific templates |
| **Compliance** | CIS Benchmarks, DISA STIG, PCI DSS, HIPAA — driven by [[definitions/cve_and_oval#oval--open-vulnerability-assessment-language]] definitions |

### Basic Network Scan — Key Settings

| Tab | Option | Notes |
|-----|--------|-------|
| Discovery | Scan fragile devices | Disable on production networks |
| Discovery | Port range | Common / All / Custom range |
| Discovery | Probe all ports for services | Enabled by default |
| Assessment | Web application scanning | Enable for web targets; set custom User-Agent |
| Assessment | Brute force / RID Brute Forcing | User enumeration via RID cycling |
| Advanced | Safe checks | Enabled by default — prevents checks that could crash targets |
| Advanced | Scan rate throttle | Reduce on congested or low-bandwidth links |
| Advanced | Random scan order | Evades detection on IDS-monitored networks |

---

## Credentialed Scanning

Credentialed scans authenticate to the target and assess the actual installed patch state — more accurate than unauthenticated scans which can only infer from banners.

### Supported Authentication Methods

| Type | Methods |
|------|---------|
| **SSH (Linux/Unix)** | Password, public key, certificate |
| **Windows** | LM/NTLM hash, Kerberos, password |
| **Databases** | Oracle, PostgreSQL, DB2, MySQL, SQL Server, MongoDB, Sybase |
| **Plaintext services** | FTP, HTTP, IMAP, IPMI, Telnet |

---

## Scan Policies

Scan policies are reusable scan configurations saved as templates.

```
New Policy → select base template → configure settings/credentials/plugins → Save
```

Policies can be exported as `.nessus` XML and imported on other Nessus instances — useful for standardising scan configs across engagements.

---

## Plugin Management

Nessus uses plugins written in NASL (Nessus Attack Scripting Language). At the time of the source module there were ~146,000 plugins covering ~58,000 CVE IDs.

- Plugins are rated: **Critical / High / Medium / Low / Info**
- Ratings map to CVSS bands (see [[definitions/cvss]])
- Individual plugins or entire plugin families can be enabled/disabled in a scan policy
- Plugin rules allow suppression of known false positives per scan

---

## Exporting Scan Results

```bash
# UI export options: .pdf, .html (Executive Summary or Custom), .csv
# Raw export: .nessus (XML) or .nessus.db (XML + KB + plugin audit trail)

# nessus-report-downloader — download all formats via REST API from CLI
# https://github.com/eelsivart/nessus-report-downloader
python3 nessus_report_downloader.py \
  -H 127.0.0.1 -p 8834 \
  -u admin -P adminpass \
  -a              # download all scans
  -A pdf,csv,html # formats
```

The `.nessus` file is plain XML — parseable with standard tools:

```bash
# Extract all Critical/High findings from a .nessus export
grep -A5 'severity="[34]"' scan.nessus | grep -E 'pluginName|description'

# Pass to EyeWitness for screenshot capture of discovered web services
eyewitness --nessus scan.nessus
```

---

## Common Scanning Issues

| Problem | Cause | Fix |
|---------|-------|-----|
| All ports shown open | Firewall returning RST for all | Disable "Ping the remote host" in Advanced Scan |
| No ports shown open | Firewall dropping all probes | Same fix as above |
| Network congestion | High scan rate on low-bandwidth link | Reduce "Max Concurrent Checks Per Host" in Performance Options |
| False positives | Banner-based detection without auth | Use credentialed scan |
| Impact on fragile devices | Safe checks off | Always keep safe checks enabled unless explicitly scoped off |

**Network impact warning:** Nessus scans generate significant traffic. Use `vnstat` to measure baseline vs. scan traffic before running on production links.

```bash
# Measure network utilisation before and during scan
vnstat -l -i eth0
```

---

## CLI Alternatives

Nessus requires a web browser for most operations. For fully headless or automated vulnerability scanning, consider:

### nuclei (Recommended for CLI-first workflows)

Template-based scanner by ProjectDiscovery. 7,000+ CVE templates. No GUI dependency.

```bash
# Install
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
# or
sudo apt install nuclei -y

# Update templates
nuclei -update-templates

# Scan for all CVEs
nuclei -u https://target.com -t cves/

# Scan for specific CVE
nuclei -u https://target.com -t cves/2021/CVE-2021-44228.yaml

# Network scan with CVSS filter
nuclei -l targets.txt -severity critical,high
```

### nmap NSE vuln scripts

Lighter than a full VA scan; good for targeted checks mid-engagement.

```bash
# Run all vuln-category NSE scripts
nmap --script vuln -p 80,443,445 10.10.10.0/24

# Specific CVE check
nmap --script smb-vuln-ms17-010 -p 445 10.10.10.0/24

# HTTP vulnerability checks
nmap --script http-vuln* -p 80,443 10.10.10.10
```

### OpenVAS / GVM (CLI via gvm-cli)

See [[tools/enumeration/openvas]] — has native CLI support via `gvm-cli` and `gvm-tools`.

---

## Gotchas & Notes

- Nessus Essentials hard-limit: 16 hosts. IPs in the same /24 each count as 1 host; ranges still count individually.
- Plugin compilation on first run takes 15–30 minutes — plan for this before a scheduled engagement.
- Safe checks prevent Nessus from running checks that could crash a device. Never disable without explicit client approval and scoping.
- Nessus results frequently flag informational items at Critical severity (e.g., "SSL Certificate Cannot Be Trusted" gets Critical CVSS). Always validate with the CVSS calculator before reporting.

## Related Pages

- [[definitions/vulnerability_assessment]] — VA methodology driving Nessus usage
- [[definitions/cvss]] — CVSS scoring displayed in Nessus results
- [[definitions/cve_and_oval]] — CVE IDs and OVAL compliance definitions used by plugins
- [[definitions/assessment_standards]] — compliance templates (PCI, HIPAA, CIS) available in Nessus
- [[definitions/va_reporting]] — exporting Nessus results into a deliverable report
- [[tools/enumeration/openvas]] — open-source alternative
- [[tools/enumeration/nmap]] — lightweight targeted vuln checks

## Sources

- raw/vulnerability_assessment/getting_started_with_nessus.md
- raw/vulnerability_assessment/nessus_vulnerability_scanning_overview.md
- raw/vulnerability_assessment/nessus_scan.md
- raw/vulnerability_assessment/nessus_advanced_scanning.md
- raw/vulnerability_assessment/nessus_scanning_issues.md
- raw/vulnerability_assessment/working_with_nessus_scan_output.md
