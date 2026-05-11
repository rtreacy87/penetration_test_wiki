# CLAUDE.md — Penetration Testing Wiki Schema

This file defines the structure, conventions, and workflows for this Obsidian-based penetration testing wiki. You are the LLM maintainer. Read this file at the start of every session. Do not modify it without explicit instruction.

---

## Core idea

This wiki is a **persistent, compounding knowledge base** for offensive security. You maintain it. The human curates sources and asks questions. You do the cross-referencing, filing, summarizing, and bookkeeping. The human never has to write wiki pages — that is your job.

The wiki mirrors the structure of an actual engagement: reconnaissance first, then enumeration, then exploitation, supported by tools and wordlists at every stage. Pages link to each other the way techniques link in the field.

---

## Directory structure

```
wiki/
├── CLAUDE.md                  ← this file (schema + instructions)
├── index.md                   ← master content catalog (you maintain this)
├── log.md                     ← append-only operation log (you maintain this)
│
├── recon/                     ← passive & active reconnaissance
│   ├── _overview.md           ← what recon is, when to use each sub-technique
│   ├── osint/                 ← passive, no direct target contact
│   │   ├── domain_information.md
│   │   ├── cloud_resources.md
│   │   ├── staff_enumeration.md
│   │   └── ...
│   └── active/                ← active scanning, DNS brute-force, etc.
│       └── ...
│
├── enumeration/               ← service/host enumeration (active, post-recon)
│   ├── _overview.md
│   ├── dns.md
│   ├── ftp.md
│   ├── smtp.md
│   ├── smb.md
│   ├── snmp.md
│   ├── nfs.md
│   ├── imap_pop3.md
│   ├── ipmi.md
│   ├── mssql.md
│   ├── mysql.md
│   ├── oracle_tns.md
│   └── ...
│
├── attack/                    ← exploitation techniques by category
│   ├── _overview.md
│   ├── brute_force.md
│   ├── password_spraying.md
│   ├── privilege_escalation.md
│   ├── lateral_movement.md
│   ├── web/
│   │   ├── sqli.md
│   │   ├── xss.md
│   │   ├── lfi_rfi.md
│   │   └── ...
│   ├── network/
│   │   └── ...
│   └── ...
│
├── tools/                     ← one page per tool
│   ├── _overview.md           ← tool comparison table, when to use what
│   ├── nmap.md
│   ├── gobuster.md
│   ├── ffuf.md
│   ├── enum4linux.md
│   ├── netcat.md
│   ├── metasploit.md
│   ├── hydra.md
│   ├── impacket.md
│   └── ...
│
├── wordlists/                 ← wordlist reference, not the lists themselves
│   ├── _overview.md
│   ├── seclist_structure.md   ← map of SecLists directory layout
│   ├── use_cases.md           ← which list for which job
│   └── custom_generation.md   ← crunch, cewl, etc.
│
├── protocols/                 ← protocol reference pages (linked from enumeration + attack)
│   ├── smb.md
│   ├── ldap.md
│   ├── kerberos.md
│   ├── rdp.md
│   └── ...
│
├── labs/                      ← lab write-ups, CTF notes, engagement notes
│   ├── _overview.md
│   ├── htb/
│   │   └── ...
│   └── thm/
│       └── ...
│
└── raw/                       ← immutable source documents (you read, never modify)
    ├── assets/                ← images clipped with Obsidian Web Clipper
    └── modules/               ← one subfolder per course module
        ├── footprinting/      ← HTB Academy: Footprinting module
        │   ├── host_based_enumeration_dns.md
        │   ├── host_based_enumeration_ftp.md
        │   ├── host_based_enumeration_smb.md
        │   ├── host_based_enumeration_smtp.md
        │   ├── host_based_enumeration_snmp.md
        │   ├── host_based_enumeration_nfs.md
        │   ├── host_based_enumeration_imap_pop3.md
        │   ├── host_based_enumeration_ipmi.md
        │   ├── host_based_enumeration_mssql.md
        │   ├── host_based_enumeration_mysql.md
        │   ├── host_based_enumeration_oracle_tns.md
        │   ├── infastructure_based_enumeration_cloud_resources.md
        │   ├── infastructure_based_enumeration_domain_information.md
        │   ├── infrastructure_based_enumeration_staff.md
        │   ├── introduction_enumeration_methodology.md
        │   ├── introduction_enumeration_principles.md
        │   ├── linux_remote_management_protocols.md
        │   ├── windows_remote_management_protocols.md
        │   ├── footprinting_lab_easy.md
        │   ├── footprinting_lab_medium.md
        │   └── footprinting_lab_hard.md
        ├── network_enumeration_with_nmap/  ← HTB Academy: Network Enumeration with Nmap
        │   ├── introduction_enumeration.md
        │   ├── introduction_to_nmap.md
        │   ├── host_enumeration_host_discovery.md
        │   ├── host_enumeration_host_and_port_scanning.md
        │   ├── host_enumeration_service_enumeration.md
        │   ├── host_enumeration_nmap_scripting_engine.md
        │   ├── host_enumeration_performance.md
        │   ├── host_enumeration_saving_the_result.md
        │   ├── bypass_security_measures_firewall_evasion.md
        │   ├── nse_scripts.md
        │   └── bypass_security_measures_[easy|medium|hard]_lab.md
        ├── linux_privilege_escalation/    ← HTB Academy: Linux Privilege Escalation
        │   ├── information_gathering_*.md (3 files)
        │   ├── environment_based_*.md (3 files)
        │   ├── permission_based_*.md (4 files)
        │   ├── service_based_*.md (7 files)
        │   ├── linux_internals_based_*.md (4 files)
        │   ├── recent_0days_*.md (4 CVEs)
        │   └── hardening_considerations_linux_hardening.md
        ├── prompt_injection_attacks/      ← HTB Academy: Prompt Injection Attacks
        │   ├── introduction_to_prompt_engineering.md
        │   ├── introduction_to_prompt_injection.md
        │   ├── direct_prompt_injection.md
        │   ├── indirect_prompt_injection.md
        │   ├── prompt_injection_reconnaissance.md
        │   ├── introduction_to_jailbreaking.md
        │   ├── jailbreaks_[I|II].md
        │   ├── tools_of_the_trade.md
        │   ├── [traditional|llm-based]_prompt_injection_mitigations.md
        │   └── skills_assessment.md
        ├── ai_evasion_sparsity/           ← HTB Academy: AI Evasion & Sparsity
        │   ├── intrduction_to_sparsity_evasion_attacks.md
        │   ├── jsma_*.md (13 files — fundamentals through aggregate analysis)
        │   ├── elasticnet*.md (11 files — fundamentals through challenge)
        │   └── skill_assessment.md
        ├── attacking_ai-application_and_systems/  ← HTB Academy: Attacking AI Applications & Systems
        │   ├── overview_of_application&_system_components.md
        │   ├── attacking_the_application_*.md (4 files)
        │   ├── attacking_the_system_*.md (3 files)
        │   └── mcp_*.md (5 files — intro through mitigations)
        └── ai_data_attacks/               ← HTB Academy: AI Data Attacks
            ├── introduction_to_ai_data.md
            ├── introduction_to_ai_data_attacks.md
            ├── introduction_to_trojan_attacks.md
            ├── label_attacks_*.md (5 files — baseline, label flipping, targeted, evaluation)
            ├── feature_attacks_*.md (5 files — baseline, clean label, target selection, attack, evaluation)
            ├── trojan_attacks_*.md (5 files — CNN arch, data prep, components, training, evaluation)
            ├── pickels_and_steganography_*.md (4 files — training, tools, attack, execute)
            ├── pickels_and_tensor_steganography.md
            └── skills_assessment.md
```

**Raw source rules:**
- Raw files are **read-only**. Never edit them. Never move them.
- When ingesting a module, read all files in that module's folder before writing any wiki pages.
- Source citations in wiki pages must use the full path from vault root: `raw/modules/footprinting/host_based_enumeration_smb.md`
- If a new module folder is added, update this CLAUDE.md to document it before ingesting.

---

## Page format

Every wiki page (except index.md and log.md) uses this structure:

```markdown
---
tags: [recon, osint, domain]        # lowercase, use existing tags when possible
module: footprinting                 # source module or course if applicable
last_updated: YYYY-MM-DD
source_count: 2                      # number of raw sources this page draws from
---

# Page Title

One-sentence summary of what this page covers.

## Overview
...

## Key concepts / techniques
...

## Commands / syntax
(code blocks for any commands)

## Flags & options
(tables work well here)

## Gotchas & notes
(edge cases, version differences, defensive measures to be aware of)

## Related pages
- [[tool-name]]
- [[enumeration/smb]]
- [[attack/brute_force]]

## Sources
- raw/modules/footprinting/host_based_enumeration_smb.md  ← full path from vault root
```

Use `[[wikilinks]]` for all internal links. Obsidian renders these as clickable links and includes them in the graph view.

---

## Tag taxonomy

Use only these top-level tag categories. Add sub-tags with a slash: `recon/osint`, `enumeration/smb`, `attack/web`.

| Tag | Use for |
|-----|---------|
| `recon` | Passive and active reconnaissance techniques |
| `enumeration` | Service/host enumeration |
| `attack` | Exploitation and post-exploitation |
| `tool` | Tool-specific pages |
| `wordlist` | Wordlist reference material |
| `protocol` | Protocol reference pages |
| `lab` | Lab write-ups and CTF notes |
| `concept` | Methodology, principles, theory |

---

## Operations

### Ingest a new module

When the human says "ingest [module name]":

1. List all files in `raw/modules/<module_name>/` to understand the scope.
2. Read each file in the module folder fully before writing anything.
3. Discuss key takeaways with the human — what's new, what contradicts existing pages, what's notable.
4. Write or update the relevant wiki pages (typically 5–15 pages per module).
5. Update `index.md`: add every new page, update one-liners for any modified pages.
6. Append a single entry to `log.md` for the whole module:
   ```
   ## [YYYY-MM-DD] ingest | <Module Name> (raw/modules/<module_name>/)
   Files read: X. Pages created: Y. Pages updated: Z.
   Key additions: [brief list of notable new content]
   ```

### Ingest a single source file

When the human drops one file into `raw/` and says "ingest this":

1. Read the file.
2. Write or update relevant wiki pages.
3. Update `index.md`.
4. Append to `log.md`:
   ```
   ## [YYYY-MM-DD] ingest | Filename
   Pages created: X. Pages updated: Y. Key additions: ...
   ```

### Answer a query

1. Read `index.md` to find relevant pages.
2. Read those pages.
3. Synthesize an answer with `[[wikilink]]` citations.
4. If the answer is substantive and reusable, offer to file it as a new wiki page.

### Add a new tool page

When a new tool is encountered in any source, create `tools/<toolname>.md` with:
- what it does
- install/usage syntax
- common flags table
- typical use cases (link to the relevant enumeration or attack page)
- example command lines

### Add a new lab write-up

Store under `labs/htb/` or `labs/thm/`. Use this structure:
- Target info (OS, difficulty, IP)
- Recon steps taken (with commands)
- Enumeration findings
- Exploitation path
- Lessons learned
- Links to relevant tool and technique pages

### Lint the wiki

When asked to lint:
1. Check for orphan pages (no inbound links).
2. Check for mentioned tools/techniques without their own page.
3. Check for stale cross-references.
4. Suggest 3–5 new pages or expansions that would add value.

---

## index.md and log.md — purpose and ownership

These two files are the wiki's navigation and audit layer. They serve different audiences.

**index.md serves the LLM.** At the start of every query, you read `index.md` first to discover which pages exist and which ones are relevant — without it you'd have to crawl the directory tree blind on every session. It's the map that makes the wiki queryable without a search engine. It also serves as a quick-scan overview for the human when they want to see what's been built. You update it on every ingest — every new page gets a line, every significantly updated page gets its one-liner revised. Never let it go stale.

**log.md serves the human.** It's the audit trail: what was ingested, when, what changed, what queries produced useful new pages. When a new session starts, the human can point you at the log so you can orient yourself without re-reading the entire wiki. It's also how the human tracks the wiki's evolution — if a page looks wrong, the log shows which ingest introduced that content. It is strictly append-only; never edit or delete past entries.

---

## index.md format

`index.md` is organized by directory. Each entry: page link, one-line summary, tags.

```markdown
# Wiki Index

_Last updated: YYYY-MM-DD — N pages total_

## Recon
- [[recon/osint/domain_information]] — Passive domain recon: WHOIS, certificate transparency, DNS history. `recon/osint`
- ...

## Enumeration
- [[enumeration/smb]] — SMB enumeration: shares, users, null sessions, smbclient, enum4linux. `enumeration` `protocol`
- ...

## Tools
- [[tools/nmap]] — Port scanning, service detection, NSE scripts. `tool`
- ...
```

---

## log.md format

Append-only. Newest entries at the top.

```markdown
# Wiki Log

## [YYYY-MM-DD] query | "How do I enumerate SMB shares without credentials?"
Synthesized from: [[enumeration/smb]], [[tools/enum4linux]], [[tools/smbclient]]

## [YYYY-MM-DD] ingest | HTB Academy — Footprinting Module
Pages created: 11. Pages updated: 3. Key additions: IPMI enumeration, Oracle TNS, IMAP/POP3.

## [YYYY-MM-DD] lint | routine health check
Orphans found: 2. New pages suggested: kerberos.md, rpcclient.md.
```

---

## Obsidian setup recommendations

These plugins and settings work well with this wiki:

- **Dataview** — query frontmatter. Example: list all `tool` pages updated in the last 30 days.
- **Graph view** — use regularly to spot orphans and identify hub pages (nmap, smb, and recon tend to be hubs).
- **Obsidian Web Clipper** — clip articles directly to `raw/assets/` as markdown.
- **Download attachments hotkey** — bind `Ctrl+Shift+D` to download inline images locally so they survive link rot.
- **Marp** — generate slide decks from wiki pages for review sessions.
- **Attachment folder** — set to `raw/assets/` in Settings → Files and links.

---

## Conventions

- File names: `lowercase_with_underscores.md`
- Wikilinks: always use the short relative path: `[[tools/nmap]]` not `[[nmap]]`
- Commands: always in fenced code blocks with the shell type: ` ```bash `
- Tables: prefer for flags, options, and comparisons
- Never duplicate content — if a technique applies to both `enumeration/smb.md` and `attack/lateral_movement.md`, write it once in the most specific location and link from the other
- Protocol pages in `protocols/` are reference-only; technique pages in `enumeration/` and `attack/` link to them

---

## Seed pages to create first

When starting from the existing module sources, create these pages in priority order:

1. `index.md` and `log.md` (scaffolding)
2. `recon/_overview.md` — high-level recon methodology
3. `enumeration/_overview.md` — enumeration layers and principles
4. One page per service in `enumeration/` (dns, ftp, smtp, smb, snmp, nfs, imap_pop3, ipmi, mssql, mysql, oracle_tns)
5. `recon/osint/domain_information.md`, `cloud_resources.md`, `staff_enumeration.md`
6. Tool pages for every tool mentioned across the modules
7. `tools/_overview.md` — comparison table
8. `wordlists/_overview.md` and `wordlists/use_cases.md`
