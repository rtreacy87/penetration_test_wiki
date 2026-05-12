---
tags: [wordlist, concept, reference]
module: multi
last_updated: 2026-05-11
source_count: 0
---

# Wordlists Overview

Reference guide for wordlist sources, installation paths, obtaining new lists, and choosing the right list for the task.

## Overview

A wordlist is only as good as its match to the target context. Generic lists waste time; targeted or mutated lists find results that generic ones miss. The workflow is always: install → locate → select for task → customize if needed.

---

## Installation Paths

### Kali Linux

| Collection           | Default Path                                      | Install / Notes                                                                                                |
| -------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| SecLists             | `/usr/share/seclists/`                            | `sudo apt install seclists` — [github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists) |
| rockyou.txt          | `/usr/share/wordlists/rockyou.txt.gz`             | Ships **gzipped** — must run `gunzip /usr/share/wordlists/rockyou.txt.gz` before use                           |
| dirbuster lists      | `/usr/share/dirbuster/wordlists/`                 | Pre-installed with dirbuster                                                                                   |
| dirb lists           | `/usr/share/dirb/wordlists/`                      | Pre-installed with dirb                                                                                        |
| wfuzz lists          | `/usr/share/wfuzz/wordlist/`                      | Pre-installed with wfuzz                                                                                       |
| Metasploit wordlists | `/usr/share/metasploit-framework/data/wordlists/` | Pre-installed with Metasploit                                                                                  |
| John the Ripper      | `/usr/share/john/password.lst`                    | Pre-installed with john                                                                                        |
| nmap NSE data        | `/usr/share/nmap/nselib/data/`                    | Pre-installed (usernames.lst, passwords.lst)                                                                   |

### Parrot OS

| Collection | Default Path | Install / Notes |
|------------|-------------|----------------|
| SecLists | `/usr/share/seclists/` | `sudo apt install seclists` — [github.com/danielmiessler/SecLists](https://github.com/danielmiessler/SecLists) |
| rockyou.txt | `/usr/share/wordlists/rockyou.txt.gz` | Ships **gzipped** — run `gunzip /usr/share/wordlists/rockyou.txt.gz` before use |
| dirbuster lists | `/usr/share/dirbuster/wordlists/` | Pre-installed with dirbuster |
| dirb lists | `/usr/share/dirb/wordlists/` | Pre-installed with dirb |
| wfuzz lists | `/usr/share/wfuzz/wordlist/` | Pre-installed with wfuzz |
| Metasploit wordlists | `/usr/share/metasploit-framework/data/wordlists/` | Pre-installed with Metasploit |

### Check what's installed and unzip rockyou

```bash
# Unzip rockyou.txt — required before first use on Kali/Parrot
gunzip /usr/share/wordlists/rockyou.txt.gz
ls -lh /usr/share/wordlists/rockyou.txt    # should be ~134 MB when unzipped

# Verify SecLists is installed
ls /usr/share/seclists/

# Find all wordlists on the system
find /usr/share -name "*.txt" -path "*/wordlist*" 2>/dev/null | head -40
find /usr/share/seclists -maxdepth 2 -type d    # SecLists directory tree
```

---

## Obtaining Wordlists from the Web

### Primary Collections

| Source | GitHub / URL | What It Contains |
|--------|-------------|-----------------|
| **SecLists** | https://github.com/danielmiessler/SecLists | The gold standard: usernames, passwords, discovery, fuzzing, SNMP, DNS — everything |
| **PayloadsAllTheThings** | https://github.com/swisskyrepo/PayloadsAllTheThings | Payloads + wordlists by vulnerability type (SQLi, SSRF, XXE, LFI, etc.) |
| **FuzzDB** | https://github.com/fuzzdb-project/fuzzdb | Attack patterns, discovery lists, predictable resource locators |
| **Assetnote Wordlists** | https://github.com/assetnote/wordlists | Auto-generated web tech lists (APIs, frameworks, extensions) — updated regularly |
| **Probable Wordlists** | https://github.com/berzerk0/Probable-Wordlists | Password lists ranked by real-world probability |

### Password-Specific Sources

| Source | GitHub / URL | What It Contains |
|--------|-------------|-----------------|
| **rockyou.txt** | https://github.com/danielmiessler/SecLists (in `Passwords/Leaked-Databases/`) | 14M passwords from the 2009 RockYou breach; the standard brute-force list |
| **Rockyou2021** | https://github.com/ohmybahgosh/RockYou2021.txt | 8.4B passwords combined from multiple breaches; 92GB uncompressed |
| **Kaonashi** | https://github.com/kaonashi-passwords/Kaonashi | Rule-generated lists; includes NLP-based password patterns |
| **Probable Wordlists** | https://github.com/berzerk0/Probable-Wordlists | Ranked by probability from real-world cracking sessions |
| **CrackStation** | https://crackstation.net/crackstation-wordlist-password-cracking-dictionary.htm | 1.5B word list; 15GB; for serious offline cracking |
| **WeakPass** | https://weakpass.com/wordlist | Curated lists ranked by effectiveness; includes targeted service lists |

### Download Commands

```bash
# ── rockyou.txt ────────────────────────────────────────────────────────────────
# Option 1: unzip from Kali/Parrot (already on disk as .gz)
gunzip /usr/share/wordlists/rockyou.txt.gz

# Option 2: extract from SecLists apt package
sudo apt install seclists
tar -xzf /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz \
    -C /usr/share/wordlists/

# Option 3: clone SecLists and extract
git clone --depth 1 https://github.com/danielmiessler/SecLists.git /usr/share/seclists
tar -xzf /usr/share/seclists/Passwords/Leaked-Databases/rockyou.txt.tar.gz \
    -C /usr/share/wordlists/

# ── Other collections ──────────────────────────────────────────────────────────
# SecLists (full clone — large, ~1.5 GB)
git clone https://github.com/danielmiessler/SecLists.git /usr/share/seclists

# PayloadsAllTheThings
git clone https://github.com/swisskyrepo/PayloadsAllTheThings.git ~/tools/PayloadsAllTheThings

# FuzzDB
git clone https://github.com/fuzzdb-project/fuzzdb.git ~/tools/fuzzdb

# Assetnote wordlists (clone index — individual lists fetched on demand)
git clone https://github.com/assetnote/wordlists.git ~/tools/assetnote-wordlists

# Kaonashi password lists
git clone https://github.com/kaonashi-passwords/Kaonashi.git ~/tools/kaonashi

# Probable Wordlists
git clone https://github.com/berzerk0/Probable-Wordlists.git ~/tools/probable-wordlists

# Rockyou2021 (warning: 92 GB uncompressed)
git clone https://github.com/ohmybahgosh/RockYou2021.txt.git ~/tools/rockyou2021

# Update SecLists in place
cd /usr/share/seclists && git pull

# Install via apt (Kali/Parrot — installs both seclists and wordlists packages)
sudo apt update && sudo apt install -y seclists wordlists
```

---

## SecLists Directory Structure

SecLists is the primary collection. Structure:

| Directory | Key Files | Use Case |
|-----------|-----------|----------|
| `Discovery/Web-Content/` | `common.txt`, `directory-list-2.3-medium.txt`, `raft-large-*.txt` | Web directory/file brute-force |
| `Discovery/DNS/` | `subdomains-top1million-5000.txt`, `...-20000.txt`, `...-110000.txt` | DNS subdomain brute-force |
| `Discovery/SNMP/` | `common-snmp-community-strings.txt`, `snmp-onesixtyone.txt` | SNMP community string brute-force |
| `Usernames/` | `top-usernames-shortlist.txt`, `xato-net-10-million-usernames.txt` | User enumeration, credential attacks |
| `Passwords/Leaked-Databases/` | `rockyou.txt.tar.gz` (extract before use) | General password brute-force and cracking |
| `Passwords/Common-Credentials/` | `top-passwords-shortlist.txt`, `10-million-password-list-top-100.txt` | Password spray |
| `Passwords/Default-Credentials/` | `default-passwords.csv`, various service-specific files | Default credential checks |
| `Fuzzing/` | `special-chars.txt`, `LFI-Jhaddix.txt`, various | Injection and fuzzing |
| `Miscellaneous/` | `user-agents.txt` | HTTP headers, user agents |

```bash
# Explore SecLists structure
tree /usr/share/seclists -L 2

# Find a specific type of list
find /usr/share/seclists -name "*ftp*" -o -name "*ssh*" | sort
find /usr/share/seclists/Passwords/Default-Credentials -type f | sort
```

---

## Wordlist Selection by Task

| Task | Recommended List | Path |
|------|-----------------|------|
| Web directory scan (fast) | `common.txt` | `Discovery/Web-Content/common.txt` |
| Web directory scan (standard) | `directory-list-2.3-medium.txt` | `Discovery/Web-Content/` |
| Web directory scan (thorough) | `raft-large-directories.txt` | `Discovery/Web-Content/` |
| Subdomain brute-force (fast) | `subdomains-top1million-5000.txt` | `Discovery/DNS/` |
| Subdomain brute-force (thorough) | `subdomains-top1million-110000.txt` | `Discovery/DNS/` |
| SNMP community strings | `common-snmp-community-strings.txt` | `Discovery/SNMP/` |
| Username enumeration | `xato-net-10-million-usernames.txt` | `Usernames/` |
| Password spray (avoid lockout) | `top-passwords-shortlist.txt` | `Passwords/Common-Credentials/` |
| General brute-force | `rockyou.txt` | `Passwords/Leaked-Databases/` |
| Default credentials | service-specific files | `Passwords/Default-Credentials/` |
| FTP/SSH usernames | `top-usernames-shortlist.txt` | `Usernames/` |

---

## Custom Wordlist Generation

### CeWL — website wordlist

```bash
# Scrape target website for custom words (depth 2, min length 5)
cewl http://target.com -d 2 -m 5 -w custom.txt

# Include email addresses found on site
cewl http://target.com -d 3 -m 5 --email -w custom_with_emails.txt
```

### Crunch — pattern-based generation

```bash
# Generate 8-char passwords: lowercase + digits
crunch 8 8 abcdefghijklmnopqrstuvwxyz0123456789 -o output.txt

# Pattern: Company + 4 digits (@ = lowercase, % = digit, ^ = uppercase)
crunch 11 11 -t Company%%%% -o company_passwords.txt

# Generate all 4-digit PINs
crunch 4 4 0123456789 -o pins.txt
```

### Hashcat rules — mutate existing lists

```bash
# Apply best64 rules to rockyou (Kali/Parrot path)
hashcat --stdout /usr/share/wordlists/rockyou.txt \
        -r /usr/share/hashcat/rules/best64.rule > mutated.txt

# Combine multiple rules
hashcat --stdout rockyou.txt \
        -r /usr/share/hashcat/rules/best64.rule \
        -r /usr/share/hashcat/rules/toggles1.rule > mutated2.txt

# List available Hashcat rules
ls /usr/share/hashcat/rules/
```

### Username generation from employee names

```bash
# username-anarchy — generates format variants (first.last, f.last, flast, etc.)
username-anarchy -f names.txt > usernames.txt

# With a single name
username-anarchy "John Smith" > john_variants.txt
```

---

## Related Pages

- [[wordlists/use_cases]] — per-tool and per-service wordlist selection
- [[enumeration/_overview]]
- [[tools/enumeration/nmap]]
- [[tools/attack/hydra]]
- [[tools/gobuster]]
- [[tools/ffuf]]
