---
tags: [definition, concept, reference]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 1
---

# VA Reporting

How to structure a vulnerability assessment or pentest report so that both executives and engineers can act on it. The report is the deliverable — everything else is evidence collection.

## Overview

Automated scanner output is not a report. Raw Nessus or OpenVAS exports contain false positives, duplicate entries, and no business context. The assessor's job is to transform raw data into a clear, validated, prioritized document that a non-technical executive can brief from and a sysadmin can work from on the same day.

## Key Report Sections

### 1. Executive Summary

**Audience:** C-suite, legal, business owners — people who will not read technical details.

**Must contain:**
- High-level narrative of overall security posture
- Total vulnerability count broken down by severity (graph or table)
- Top 3–5 most critical risks with one-sentence business impact each
- Recommended next step (patch immediately, remediation roadmap, or re-test)

Keep it to 1–2 pages maximum.

---

### 2. Overview of Assessment

**Audience:** Technical managers and compliance reviewers.

**Must contain:**
- Methodology used (PTES, NIST, etc. — see [[definitions/assessment_standards]])
- Tools and techniques used (scanner version, plugin counts, credential scope)
- Summary of how the assessment was executed
- Any deviations from planned scope or methodology

---

### 3. Scope and Duration

**Must contain:**
- All IP ranges, hostnames, URLs, and cloud accounts authorized for testing
- Exact start and end dates/times of testing
- Any explicitly excluded assets or test types
- Rules of engagement (credentialed vs. unauthenticated, destructive vs. safe-checks-only)

Scope and duration sections are legally significant — they define the boundary of what was tested and what wasn't.

---

### 4. Vulnerabilities and Recommendations

**Audience:** System administrators, developers, security engineers.

Each finding entry must include:

| Field | Content |
|-------|---------|
| **Vulnerability Name** | Descriptive title (match CVE title where possible) |
| **CVE** | CVE ID (or `N/A` for non-CVE findings) |
| **CVSS Score** | v3.1 Base Score and severity band |
| **Description** | What the vulnerability is and why it matters |
| **References** | NVD link, vendor advisory, PoC URL |
| **Remediation Steps** | Specific, actionable fix — not just "update the software" |
| **Proof of Concept** | Screenshot, output, or PoC command demonstrating exploitability |
| **Affected Systems** | IP, hostname, service, version |

Order findings by CVSS score descending. Group by host or by vulnerability class — choose based on what makes remediation easier for the client.

---

## Exporting from Scanners

### Nessus Export Formats

```bash
# Download reports via Nessus REST API (nessus-report-downloader)
python3 nessus_report_downloader.py -H 127.0.0.1 -p 8834 \
  -u admin -P adminpass -a -A pdf,csv,html

# Export formats: .pdf, .html (Executive Summary or Custom), .csv (column-selectable)
# Raw export: .nessus (XML + settings) or .nessus.db (XML + KB + audit trail)
```

### OpenVAS Export Formats

```bash
# Export from CLI using gvm-cli
gvm-cli --gmp-username admin --gmp-password admin socket \
  --xml "<get_reports report_id='<UUID>' format_id='c402cc3e-b531...' />"

# openvasreporting: convert XML to Excel
pip install openvasreporting
python3 -m openvasreporting -i report.xml -f xlsx
```

Available OpenVAS formats: XML, CSV, PDF, ITG, TXT, XLSX (via openvasreporting).

---

## Writing Tips

- **Validate every Critical and High before reporting it.** A false positive in the report destroys credibility.
- **Remediation steps must be specific.** "Apply the vendor patch" is not actionable — give the patch KB number, the config change, or the command.
- **Proof of concept matters.** A screenshot showing the actual scan output or a working exploit command is more convincing than a description.
- **Match tone to audience.** Executive Summary gets business-impact language; technical sections get command-line output.
- **Consistent CVSS.** If you manually adjusted a CVSS score from the scanner default (e.g., because the asset is isolated), document why.

## Gotchas & Notes

- The reporting section is the most visible deliverable — a technically brilliant assessment with a confusing report will not result in remediation.
- Clients often share reports with their own auditors — scope and duration sections will be scrutinized for coverage gaps.
- Never include raw flags or pentest infrastructure details in client-facing reports.

## Related Pages

- [[definitions/vulnerability_assessment]] — VA methodology that produces the findings
- [[definitions/cvss]] — CVSS scoring used in each finding entry
- [[definitions/cve_and_oval]] — CVE IDs and OVAL definitions referenced in findings
- [[definitions/assessment_standards]] — frameworks that define required report sections for compliance
- [[tools/enumeration/nessus]] — Nessus scan output and export options
- [[tools/enumeration/openvas]] — OpenVAS report export and openvasreporting tool

## Sources

- raw/vulnerability_assessment/reporting.md
- raw/vulnerability_assessment/working_with_nessus_scan_output.md
- raw/vulnerability_assessment/openvas_exporting_the_results.md
