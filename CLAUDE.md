# CLAUDE.md — Penetration Testing Wiki Schema

This file defines the structure, conventions, and workflows for this Obsidian-based penetration testing wiki. You are the LLM maintainer. Read this file at the start of every session. Do not modify it without explicit instruction.

---

## Core idea

This wiki is a **persistent, compounding knowledge base** for offensive security. You maintain it. The human curates sources and asks questions. You do the cross-referencing, filing, summarizing, and bookkeeping. The human never has to write wiki pages — that is your job.

The wiki mirrors the structure of an actual engagement: reconnaissance first, then enumeration, then exploitation, supported by tools and wordlists at every stage. Pages link to each other the way techniques link in the field.

---

## Directory structure

The tree below is intentionally abstract — it describes folder purposes and naming conventions, not individual files. For the current page inventory, read `index.md`. Never update this section just because a new page or module was added.

```
wiki/
├── CLAUDE.md                  ← this file (schema + instructions)
├── index.md                   ← master content catalog (you maintain this)
│
├── recon/                     ← passive & active reconnaissance
│   ├── _overview.md
│   ├── osint/                 ← one page per OSINT technique (passive, no target contact)
│   └── active/                ← one page per active recon technique
│
├── enumeration/               ← service/host enumeration (active, post-recon)
│   ├── _overview.md
│   └── <service>.md           ← one page per protocol or service
│
├── attack/                    ← exploitation techniques by category
│   ├── _overview.md
│   ├── <technique>.md         ← flat technique pages (smb, dns, rdp, sql, etc.)
│   ├── ai/                    ← AI/ML-specific attack techniques
│   ├── network/               ← network-layer attacks (firewall evasion, etc.)
│   └── web/                   ← web application attacks (sqli, xss, lfi, etc.)
│
├── tools/                     ← one page per tool; split into three subfolders
│   ├── _overview.md           ← tool comparison table, when to use what
│   ├── utility/               ← general-purpose clients and runtimes
│   ├── enumeration/           ← tools primarily used for discovery
│   └── attack/                ← tools primarily used for exploitation
│
├── wordlists/                 ← wordlist reference (not the lists themselves)
│   └── _overview.md
│
├── definitions/               ← pentest-oriented concept definitions
│   └── _overview.md
│
├── shell_commands/            ← quick-reference command sheets by shell environment
│
├── ports/                     ← port reference tables
│
├── protocols/                 ← protocol reference pages (linked from enumeration + attack)
│
├── labs/                      ← publishable lab write-ups and CTF notes
│   ├── _overview.md
│   ├── htb/
│   │   └── <module>/          ← one subfolder per HTB module
│   └── thm/
│
└── raw/                       ← immutable source documents (you read, never modify)
    ├── assets/                ← images clipped with Obsidian Web Clipper
    ├── lab/                   ← GITIGNORED — private lab solutions, never linked from wiki pages
    │   └── <module>/          ← name must match labs/htb/<module>/
    └── <module_name>/         ← one subfolder per source module
                                  run `ls raw/` to discover available modules
                                  run `ls raw/<module>/` before ingesting to see its files
```

**Raw source rules:**
- Raw files are **read-only**. Never edit them. Never move them.
- Before ingesting a module, `ls raw/<module_name>/` to see its files — do not guess filenames.
- When ingesting, read all files in the module folder before writing any wiki pages.
- Source citations in wiki pages must use the full path from vault root: `raw/modules/footprinting/host_based_enumeration_smb.md`
- **Do not update this CLAUDE.md when new modules or pages are added.** `index.md` owns that record.

---

## Page format

Every wiki page (except index.md) uses this structure:

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
| `definition` | Terminology, protocol definitions, concept glossaries |
| `shell` | Shell command references (bash, cmd, PowerShell) |
| `reference` | Quick-reference tables (ports, flags, syntax) |

---

## Operations

### Ingest a new module

When the human says "ingest [module name]":

1. List all files in `raw/modules/<module_name>/` to understand the scope.
2. Read each file in the module folder fully before writing anything.
3. Discuss key takeaways with the human — what's new, what contradicts existing pages, what's notable.
4. Write or update the relevant wiki pages (typically 5–15 pages per module).
5. Update `index.md`: add every new page, update one-liners for any modified pages.

### Ingest a single source file

When the human drops one file into `raw/` and says "ingest this":

1. Read the file.
2. Write or update relevant wiki pages.
3. Update `index.md`.

### Answer a query

1. Read `index.md` to find relevant pages.
2. Read those pages.
3. Synthesize an answer with `[[wikilink]]` citations.
4. If the answer is substantive and reusable, offer to file it as a new wiki page.

### Add a new tool page

Tools are stored in three subfolders under `tools/`:
- `tools/utility/` — general-purpose clients and frameworks (sqlcmd, impacket, legacy tools)
- `tools/enumeration/` — tools primarily for discovery (nmap, smbclient, dig, linpeas, pspy, etc.)
- `tools/attack/` — tools primarily for exploitation (netexec, metasploit, responder, hydra, etc.)

When a new tool is encountered in any source, create `tools/<category>/<toolname>.md` with:
- what it does (one-sentence summary + Overview section)
- `## Installation` section (see format below — mandatory on all tool pages)
- usage/syntax with examples
- common flags table
- typical use cases (link to the relevant enumeration or attack page)
- `## Gotchas & Notes` for edge cases
- `## Related Pages` wikilinks

**Installation section format (Kali / Parrot):**
```markdown
## Installation

```bash
# Check if installed
<command --version or which command>

# Install (Kali / Parrot)
sudo apt install <package> -y

# Verify
<command --version>
```
```

Additional rules:
- For tools that are deprecated/legacy (e.g., crackmapexec), include removal instructions and point to the replacement
- For tools that run on the TARGET rather than the attacker (e.g., linpeas, pspy), note this and show the download/staging commands instead of apt install
- For tools requiring external repos (e.g., sqlcmd via Microsoft repo), show the full repo-add + install sequence
- Assume Kali or Parrot OS. Do not include macOS or Windows install instructions unless the tool is Windows-only

### Add a new lab write-up

**Directory convention:** Lab write-ups are stored under `labs/htb/<module>/` where `<module>` matches the subfolder name in `raw/lab/<module>/` (if a private solution file exists). If no `raw/lab/<module>/` folder exists for that lab, store directly under `labs/htb/`.

Example: `raw/lab/attacking_common_services/attacking_sql_lab.md` → write-up at `labs/htb/attacking_common_services/<labname>.md`

**Private solution files (`raw/lab/`):**
- `raw/lab/` is gitignored and never published
- When a `raw/lab/<module>/` file exists for the lab being written, read it for context (flags, exact answers, step details) to inform the write-up
- Never link to `raw/lab/` files from any wiki page — they are not part of the published wiki
- Never include raw flags or verbatim answers in the published write-up under `labs/htb/`

**Write-up structure:**
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

## index.md — purpose and ownership

**index.md serves the LLM.** At the start of every query, you read `index.md` first to discover which pages exist and which ones are relevant — without it you'd have to crawl the directory tree blind on every session. It's the map that makes the wiki queryable without a search engine. It also serves as a quick-scan overview for the human when they want to see what's been built. You update it on every ingest — every new page gets a line, every significantly updated page gets its one-liner revised. Never let it go stale.

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
- [[tools/enumeration/nmap]] — Port scanning, service detection, NSE scripts. `tool`
- ...
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
- Wikilinks: always use the short relative path: `[[tools/enumeration/nmap]]` not `[[nmap]]`
- Commands: always in fenced code blocks with the shell type: ` ```bash `
- Tables: prefer for flags, options, and comparisons
- Never duplicate content — if a technique applies to both `enumeration/smb.md` and `attack/lateral_movement.md`, write it once in the most specific location and link from the other
- Protocol pages in `protocols/` are reference-only; technique pages in `enumeration/` and `attack/` link to them
- Definition pages explain *what something is* and *why a pentester cares*; keep them factual and cross-link to the enumeration/attack pages that use the concept
- Shell command pages are quick-reference sheets, not tutorials; include the command, a one-line description, and a brief example — no prose explanations
- The ports page is a lookup table; include port, protocol (TCP/UDP), service, and a "pentester's interest" column

