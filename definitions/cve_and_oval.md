---
tags: [definition, concept, reference]
module: vulnerability_assessment
last_updated: 2026-05-21
source_count: 1
---

# CVE and OVAL

CVE (Common Vulnerabilities and Exposures) is the universal identifier for publicly disclosed vulnerabilities. OVAL (Open Vulnerability Assessment Language) is the XML-based standard for describing how to detect them. Together they form the backbone of automated vulnerability scanning.

## CVE — Common Vulnerabilities and Exposures

### What it is

A publicly available catalog of security issues sponsored by the U.S. Department of Homeland Security (DHS) and maintained by MITRE. Every entry gets a unique CVE ID assigned by a CVE Numbering Authority (CNA).

**ID format:** `CVE-YYYY-NNNNN` (year + sequence number, e.g., `CVE-2021-44228`)

### Eligibility Requirements

To receive a CVE, a vulnerability must:
1. Be independently fixable (a standalone issue, not a design decision)
2. Affect only one codebase
3. Be acknowledged and documented by the relevant vendor

### Stages of Obtaining a CVE

| Stage | Action |
|-------|--------|
| 1 | Identify if CVE is required and relevant |
| 2 | Reach out to the affected product vendor |
| 3 | Determine whether to contact vendor CNA or a third-party CNA |
| 4 | Submit CVE request via the CVE web form |
| 5 | Receive confirmation of submission |
| 6 | Receive CVE ID (may be in RESERVED state until disclosure) |
| 7 | Public disclosure of CVE ID and details |
| 8 | Announce the CVE to relevant communities |
| 9 | Provide full technical details to the CVE Team |

### Responsible Disclosure

Security researchers should work directly with the vendor *before* publishing — provide full technical details so a patch can be developed and released before public announcement. The coordinated disclosure window is typically 90 days (Google Project Zero standard).

**Why this matters operationally:** During an engagement, if you discover an unpatched 0-day in client infrastructure, document it privately and advise responsible disclosure rather than publishing immediately.

### Notable CVE Examples

| CVE | Name | Impact |
|-----|------|--------|
| CVE-2020-5902 | BIG-IP TMUI RCE | Unauthenticated RCE in F5 BIG-IP traffic management UI; complete system takeover |
| CVE-2021-34527 | PrintNightmare | RCE via Windows Print Spooler; remote and local exploitation, full system takeover |
| CVE-2021-44228 | Log4Shell | RCE via JNDI injection in Apache Log4j; affected thousands of Java applications |
| CVE-2019-0708 | BlueKeep | RCE via RDP on Windows 7/2008; pre-auth, no interaction required |

### Searching CVEs

```bash
# searchsploit — search Exploit-DB by CVE
searchsploit CVE-2021-34527

# nuclei — run a specific CVE template
nuclei -u https://target -t cves/2021/CVE-2021-44228.yaml

# look up CVSS and details
curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2021-34527" | jq '.vulnerabilities[0].cve.metrics'
```

---

## OVAL — Open Vulnerability Assessment Language

### What it is

An international information security standard (co-supported by U.S. DHS) for describing system state and security issues in machine-readable XML. The OVAL repository contains 7,000+ public definitions. OVAL is a core component of NIST's Security Content Automation Protocol (SCAP).

### OVAL Three-Step Process

1. **Identify** — describe the system configuration items to check (registry keys, file versions, process states)
2. **Evaluate** — compare the current system state against the OVAL definition
3. **Report** — output results in a structured, machine-readable format

### OVAL Definition Classes

| Class | Purpose |
|-------|---------|
| **Vulnerability Definitions** | Does this specific CVE affect this system? |
| **Compliance Definitions** | Does the system meet a specific configuration policy? |
| **Inventory Definitions** | Is a specific product installed on this system? |
| **Patch Definitions** | Has a specific patch been applied? |

### Relation to Scanners

Nessus and OpenVAS both use OVAL-derived logic for compliance scanning templates. When you run a Nessus CIS Benchmark scan or a DISA STIG compliance audit, OVAL definitions are driving the checks under the hood.

```xml
<!-- Example OVAL definition structure (simplified) -->
<oval_definitions>
  <definition id="oval:org.mitre.oval:def:12345" class="vulnerability">
    <metadata>
      <title>CVE-2021-34527 — Windows Print Spooler RCE</title>
      <reference ref_id="CVE-2021-34527" source="CVE"/>
    </metadata>
    <criteria>
      <criterion test_ref="oval:...registry_test"/>
    </criteria>
  </definition>
</oval_definitions>
```

## Gotchas & Notes

- A CVE in RESERVED state means the ID was allocated but details aren't public yet — often seen in coordinated disclosure windows.
- CNAs include major vendors (Microsoft, Google, Red Hat) who assign CVEs for their own products. MITRE is the CNA of last resort.
- OVAL definitions are version-specific — a definition for Windows 10 21H2 will not match Windows 10 22H2 unless the vendor published separate definitions.
- Not all vulnerabilities have CVEs: configuration issues, logic flaws, and 0-days often don't. During an engagement, document non-CVE findings with a manual CVSS assessment.

## Related Pages

- [[definitions/cvss]] — severity scoring tied to each CVE
- [[definitions/security_terminology]] — brief CVE/CVSS overview in context of other terms
- [[definitions/vulnerability_assessment]] — VA methodology that drives CVE-based scanning
- [[tools/enumeration/nessus]] — uses NASL plugins mapped to CVE IDs
- [[tools/enumeration/openvas]] — uses NVTs mapped to CVE IDs

## Sources

- raw/vulnerability_assessment/common_vulnerabilities_and_exposures.md
