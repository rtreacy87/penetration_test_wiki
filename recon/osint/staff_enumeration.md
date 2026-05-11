---
tags: [recon, recon/osint]
module: footprinting
last_updated: 2026-05-10
source_count: 1
---

# Staff Enumeration

Finding employee information via LinkedIn, job postings, and GitHub to infer technology stack, email patterns, and potential attack vectors.

## Overview

Staff enumeration is high-value passive recon that reveals the human infrastructure behind a company: who works there, what they know, what they build with, and what they inadvertently expose.

Employees are sources of intelligence in two directions:
1. What they publicly share reveals the company's technology stack, tooling, and processes.
2. They are themselves potential attack targets (phishing, password spraying, credential stuffing).

## Key Concepts / Techniques

### LinkedIn

LinkedIn is the primary source for staff enumeration. Key search dimensions:
- **Employees at a target company** — find technical staff (developers, security engineers, sysadmins)
- **Job titles** — "Software Engineer", "DevOps", "Security Analyst" indicate what infrastructure exists
- **Skills listed** — reveals technology stack (React, Django, PostgreSQL, Kubernetes, etc.)
- **Career history** — older roles and projects may reveal legacy systems still in use

**LinkedIn search filters**: connections, location, company, school, industry, language, services, title.

### Job Postings

Job postings are intentional disclosures of technology. Companies must describe their environment to attract candidates. Extract:
- Programming languages (Java, Python, C#, PHP, Ruby)
- Databases (PostgreSQL, MySQL, Oracle, SQL Server)
- Frameworks (Django, Flask, Spring, ASP.NET)
- Infrastructure tools (Docker, Kubernetes, Git, Jenkins, Terraform)
- Third-party services (Atlassian suite, Elastic, Redis, Kafka)
- Security tooling (mentions of WAF, SIEM, specific standards)

Example from source:
```
Required: Java, C#, C++, Python, Ruby, PHP, Perl
Databases: PostgreSQL, MySQL, SQL Server, Oracle
Frameworks: Flask, Django, Spring, ASP.NET MVC
Services: REST APIs, GitHub, SVN, Atlassian Suite (Confluence, Jira, Bitbucket)
```

This level of detail is a complete technology fingerprint.

### GitHub and Code Repositories

Employees who publish open-source work often expose:
- Personal email addresses in git commit history
- Hard-coded credentials, API keys, JWT tokens
- Internal tooling and configuration examples
- Security misconfigurations following publicly available tutorials

Search GitHub for the company name, domain, and employee usernames. Check commit history and repository contents carefully.

### Deriving Email Patterns

Once you know employee names, derive the email pattern from any confirmed email (support tickets, press releases, LinkedIn profile, GitHub commits). Common patterns:
- `firstname.lastname@company.com`
- `f.lastname@company.com`
- `flastname@company.com`
- `firstname@company.com`

Confirm the pattern from any known-good email, then generate the full staff list.

### Xing

Xing is a European professional network (similar to LinkedIn) commonly used in German-speaking countries. Worth checking for targets with European presence.

## Commands / Syntax

```bash
# Search for email pattern exposure in GitHub
# (do this via browser search: site:github.com "company.com" password)

# theHarvester - automated email/employee discovery
theHarvester -d inlanefreight.com -b linkedin,google,bing

# Hunter.io API for email pattern discovery
curl "https://api.hunter.io/v2/domain-search?domain=inlanefreight.com&api_key=YOURKEY"
```

## Flags & Options

| Source | What to Extract |
|--------|----------------|
| LinkedIn profiles | Name, title, skills, previous employers, education, publications |
| Job postings | Tech stack, frameworks, databases, third-party services |
| GitHub commits | Email addresses, credential exposure, configuration files |
| Press releases | Contact names and email addresses |
| Forum posts | Employee discussion of internal systems (credentials, SSH key discussion) |

## Gotchas & Notes

- Job postings are the most underutilized recon source. A single post often completely fingerprints the technology stack.
- Employees posting about internal systems on external forums (Stack Overflow, Reddit, GitHub issues) often include sensitive configuration details.
- JWT tokens found in GitHub repositories are often still valid — test immediately.
- Django and other framework security misconfigurations are well-documented. Knowing a company uses Django narrows the attack surface significantly.
- Focus on developers and security staff. Developers expose the stack; security staff reveal the defensive posture.
- Email patterns derived from staff enumeration enable targeted password spraying and phishing, but confirm the pattern with at least 2–3 known-good emails before building a full list.

## Related Pages

- [[recon/_overview]]
- [[recon/osint/domain_information]]
- [[recon/osint/cloud_resources]]

## Sources

- raw/footprinting/infrastructure_based_enumeration_staff.md
