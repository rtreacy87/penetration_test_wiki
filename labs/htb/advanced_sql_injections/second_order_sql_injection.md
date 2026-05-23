---
tags: [lab, attack/web]
module: advanced_sql_injections
last_updated: 2026-05-22
---

# Lab: Second-Order SQL Injection

Target: BlueBird application on port 8080. Register an account at `/signup` before starting.

This lab exploits a second-order SQLi where the payload is stored via `POST /profile/edit` and triggered when `GET /profile/{id}` renders the profile page. The injection can be done with curl or a browser.

## Tool Installation

```bash
# curl (pre-installed on Kali)
curl --version || sudo apt install curl -y
```

No GUI is required. A browser is an alternative to curl for the form interaction, but curl is fully practical.

---

## Question 1 — Password hash of user `betrayedApples3`

**Goal:** Exploit the second-order SQLi to exfiltrate the `password` column of the user `betrayedApples3`.

### Step 1 — Understand what makes this a second-order vulnerability

**What a tester looks for in profile editing functionality:** Any feature that lets you update stored data and then later displays that data is a candidate for second-order injection. The danger is that input sanitization at the storage stage (POST /profile/edit) may seem fine — the value is stored as-is, no SQL error. The injection doesn't execute until the data is *retrieved* by a subsequent query, where it gets concatenated unsafely.

**Why WAFs and naive input filters miss this:** A WAF scanning POST /profile/edit sees `' UNION SELECT ...` and might block it. But many WAFs also have "application mode" configurations that trust stored data — the assumption being that data already in the database must be safe. Second-order injection breaks this assumption.

**From source review:** Two endpoints are involved:
- `POST /profile/edit` — stores the user-supplied `email` value in the database. This is the *source* (where the payload is stored).
- `GET /profile/{id}` — retrieves the user record and builds a query by concatenating the stored email without sanitization. This is the *sink* (where execution happens).

### Step 2 — Register an account and understand what you can edit

**Why you need an account:** The profile edit endpoint is authenticated. Before you can inject a payload, you need to be logged in. Create an account at `/signup` with any data — the credentials only need to be valid, not meaningful.

```bash
curl -s -c cookies.txt -X POST "http://<TARGET_IP>:8080/signup" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=attacker&password=P@ssw0rd123&email=test@test.com&name=Test"

curl -s -c cookies.txt -X POST "http://<TARGET_IP>:8080/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=attacker&password=P@ssw0rd123"
```

**What to look for after login:** Note your user ID. Visit your profile page at `/profile/<your_id>` and observe which fields are displayed. The fields the page renders are the columns returned by the GET /profile/{id} query — and those are the positions you can exploit with a UNION to inject your own data.

### Step 3 — Determine the UNION structure

**What you need to know before sending the UNION payload:** The number of columns in the original SELECT and which position corresponds to the visually rendered field. Without source code, you'd need to probe this iteratively: try `UNION SELECT NULL, NULL, NULL--` with increasing NULLs until you stop getting an error, then identify which column appears where on the page. With source code, you can read the SELECT directly.

From `GET /profile/{id}` in the source: the query returns 5 columns in a specific order. The `email` column is position 4. A UNION payload that puts the target's password hash in position 4 will render it where the email normally displays on the profile page.

**Decision point — UNION vs blind:** UNION is the fastest path here because the profile page renders the result visually. Blind extraction would take 7+ requests per character of a 60-character bcrypt hash — 420+ requests versus 1. Always use the fastest available channel.

### Step 4 — Inject the UNION payload via profile edit

**Why the HTML frontend blocks this without DevTools or curl:** The email field has `type="email"`, which triggers HTML5 browser-side format validation. A UNION payload isn't a valid email format, so a browser submitting the form would reject it before the request even leaves. This is client-side-only validation — the server does not independently validate email format.

**Two ways to bypass client-side validation:**

Using curl (bypasses the browser entirely):

```bash
curl -s -b cookies.txt -X POST "http://<TARGET_IP>:8080/profile/edit" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "email=' UNION SELECT '1','2','3',password,5 FROM users WHERE username='betrayedApples3'--"
```

Using the browser with DevTools: Open the profile edit page, open DevTools (F12) → Elements, locate the email `<input>` element, change `type="email"` to `type="text"`, paste the payload in the field, and submit.

**Why curl is preferred here:** Curl avoids any frontend state issues and is repeatable. If you need to retry the payload (different column count, wrong column position), curl makes that fast. DevTools changes reset when you reload the page.

**What success looks like:** The application stores the payload in the email column without complaint. The SQL isn't evaluated yet — it's just text in the database at this stage. A success response (redirect to profile, or "Profile updated" message) confirms storage.

### Step 5 — Trigger the injection by visiting your profile page

**Why you visit YOUR profile, not the target's:** The UNION payload was stored in your account's email field. When you visit your own profile page, the backend runs `SELECT ... FROM users WHERE id = <your_id>`, which retrieves your record including the stored payload as the email value. That value then gets concatenated into a second query — this is where the injection executes.

```bash
curl -s -b cookies.txt "http://<TARGET_IP>:8080/profile/<YOUR_USER_ID>"
```

The page response will show the UNION result instead of a normal profile. The password hash of `betrayedApples3` appears in the position where your email address would normally be displayed.

---

## How the UNION Payload Works

| Part | Explanation |
|------|-------------|
| `'` | Closes the email string in the concatenated WHERE clause |
| `UNION SELECT` | Adds a second result row from an attacker-controlled query |
| `'1','2','3'` | Placeholder values for columns that don't need to be exfiltrated |
| `password` | The column to exfiltrate — lands in position 4 where email renders |
| `5` | Placeholder for the fifth column |
| `WHERE username='betrayedApples3'` | Filters to the target user |
| `--` | Comments out the rest of the original query |

---

## Lessons Learned

- Second-order injection is missed by input-time WAFs and naive filters because the dangerous string is stored as inert data, not executed until retrieved
- `type="email"` is a usability feature, not a security control — the server must validate independently; client-side validation is always bypassable
- UNION-based injection is the fastest second-order extraction method when you can see the rendered output — one request vs hundreds for blind techniques
- Profile edit endpoints are a classic second-order pattern: store → retrieve → render

## Related Pages

- [[attack/web/second_order_sqli]] — sink/source tracing, multi-step exploitation
- [[attack/web/white_box_sqli_methodology]] — identifying second-order injection points via grep
- [[labs/htb/advanced_sql_injections/searching_for_strings]] — finding the injectable endpoints
