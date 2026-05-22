---
tags: [tool, enumeration]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 3
---

# OpenVAS / GVM

Greenbone's open-source vulnerability scanner. OpenVAS is the scanner component; the full stack is called Greenbone Vulnerability Manager (GVM). Performs authenticated and unauthenticated network vulnerability scanning using a library of Network Vulnerability Tests (NVTs).

> **GUI-heavy tool.** The primary interface is the Greenbone Security Assistant (GSA) web app. For headless/automated workflows see the [CLI Alternatives](#cli-alternatives) section below.

## Installation

```bash
# Install GVM stack (Kali / Parrot)
sudo apt install gvm -y

# Initial setup (takes 20–30 minutes — downloads NVT feed)
sudo gvm-setup

# Verify setup and retrieve generated admin password
sudo gvm-check-setup

# Start GVM services
sudo gvm-start

# Access the GSA web interface
# Default: https://127.0.0.1:9392
# Credentials: admin / <password shown by gvm-setup>
```

```bash
# Stop services
sudo gvm-stop

# Check service status
sudo gvm-check-setup
```

---

## Configuring Targets

Before creating a scan, define targets in GSA:

```
Configurations → Targets → New Target icon
```

Per-target options:
- **Hosts:** IP, range, or hostname list
- **Ports:** common, all, or custom range
- **Authentication:** SSH (key or password), SMB (Windows), ESXi, SNMP
- **Alive detection method:** ICMP, TCP SYN, TCP ACK, ARP, consider all alive

**Authenticated scans** require a high-privilege account (root / Administrator). Credentialed scans reveal actual patch state rather than inferring from banners.

---

## Scan Configurations

OpenVAS uses NVT Families to compose scan policies. Built-in configurations:

| Config | Purpose |
|--------|---------|
| **Base** | OS/host enumeration only; no vulnerability checks |
| **Discovery** | Service, hardware, and software enumeration; no CVE checks |
| **Host Discovery** | Alive detection only; no vulnerability checks |
| **System Discovery** | OS and hardware fingerprinting |
| **Full and Fast** | Recommended default — full NVT checks with safe optimisations |

To create a custom scan:

```
Scans → Tasks → Wizard icon → configure target, scanner, scan config
```

---

## Running a Scan

```
Scans → Tasks → New Task
```

1. Set target (must be pre-configured)
2. Select scan config (Full and Fast for most engagements)
3. Set scanner (OpenVAS Default)
4. Click Save → Start

Progress is shown in the Tasks list. Click the task to view live results.

---

## Viewing and Exporting Results

Results appear in `Scans → Reports`. Each report has tabs for:
- Vulnerabilities (severity, CVE, NVT ID)
- Operating system and host info
- Open ports and services
- TLS certificates

### Export Formats

```bash
# From the GSA UI: Reports → select report → Export icon → choose format
# Formats: XML, CSV, PDF, ITG (Greenbone proprietary), TXT

# Generate Excel from XML export using openvasreporting
pip install openvasreporting
python3 -m openvasreporting -i report.xml -f xlsx

# Multiple reports into one workbook
python3 -m openvasreporting -i report1.xml report2.xml -f xlsx -o combined_report
```

---

## CLI Access via gvm-cli

`gvm-cli` is the official command-line interface for the Greenbone Management Protocol (GMP). Suitable for scripting and CI/CD integration.

```bash
# Install gvm-tools
pip install gvm-tools

# Interactive GMP shell
gvm-cli --gmp-username admin --gmp-password admin socket

# Non-interactive: list all tasks
gvm-cli --gmp-username admin --gmp-password admin socket \
  --xml "<get_tasks/>"

# List all reports
gvm-cli --gmp-username admin --gmp-password admin socket \
  --xml "<get_reports/>"

# Download a report as PDF (replace UUID with actual report ID)
gvm-cli --gmp-username admin --gmp-password admin socket \
  --xml "<get_reports report_id='UUID' format_id='c402cc3e-b531-11e1-9163-406186ea4fc5'/>" \
  > report.pdf

# Create and start a scan via Python (gvm-tools library)
python3 -c "
from gvm.connections import UnixSocketConnection
from gvm.protocols.gmp import Gmp
conn = UnixSocketConnection()
with Gmp(connection=conn) as gmp:
    gmp.authenticate('admin', 'admin')
    tasks = gmp.get_tasks()
    print(tasks)
"
```

---

## CLI Alternatives

For fully headless workflows without the GVM stack overhead:

### nuclei (Recommended for CLI-first workflows)

```bash
# Install
sudo apt install nuclei -y

# Update templates
nuclei -update-templates

# Full CVE scan
nuclei -l targets.txt -t cves/ -severity critical,high

# Specific service scan
nuclei -l targets.txt -t network/ -severity critical
```

### nmap NSE vuln scripts

```bash
# All vuln-category scripts
nmap --script vuln -p- 10.10.10.10

# SMB-specific vulnerability checks
nmap --script smb-vuln* -p 445 10.10.10.0/24

# HTTP vulnerability scan
nmap --script http-vuln* -p 80,443 10.10.10.10
```

### nikto (Web application focused)

```bash
sudo apt install nikto -y

# Basic web scan
nikto -h http://target.com

# With authentication
nikto -h http://target.com -id admin:password

# Output to file
nikto -h http://target.com -o output.html -Format html
```

---

## Gotchas & Notes

- `gvm-setup` takes 20–30 minutes on first run — it downloads the full NVT feed (~170,000+ tests). Plan accordingly before an engagement.
- The generated admin password is shown only once during `gvm-setup`. If missed, reset with: `sudo runuser -u _gvm -- gvmd --user=admin --new-password='newpass'`
- OpenVAS NVT severity ratings use CVSS — see [[definitions/cvss]] for interpretation.
- Full and Fast is the recommended config: it skips NVTs with high risk of false positives and unsafe operations that could harm targets.
- Like Nessus, authenticated scans are significantly more thorough — banner-only checks miss most patching information.
- `openvasreporting` requires the XML export (not CSV/PDF) as input.

## Related Pages

- [[definitions/vulnerability_assessment]] — VA methodology driving OpenVAS usage
- [[definitions/cvss]] — CVSS scores used in NVT severity ratings
- [[definitions/cve_and_oval]] — CVE and OVAL definitions underlying NVT checks
- [[definitions/assessment_standards]] — compliance requirements satisfied by OpenVAS scans
- [[definitions/va_reporting]] — exporting and formatting results for clients
- [[tools/enumeration/nessus]] — commercial alternative with more compliance templates
- [[tools/enumeration/nmap]] — lightweight, fast targeted checks

## Sources

- raw/vulnerability_assessment/getting_started_with_openvas.md
- raw/vulnerability_assessment/openvas_scan.md
- raw/vulnerability_assessment/openvas_exporting_the_results.md
