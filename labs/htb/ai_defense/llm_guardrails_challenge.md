---
tags: [lab, attack/ai, concept]
module: ai_defense
last_updated: 2026-05-21
source_count: 1
---

# LLM Guardrails Challenge — SnackOverflow Chatbot

Defensive implementation challenge for SnackOverflow's AI chatbot. The task is to write two guardrail functions — `input_guardrail` and `output_guardrail` — that sanitise user input before it reaches the LLM and validate/sanitise LLM output before it is returned to users. The chatbot has a plugin that can fetch content from external URLs, making both guardrail layers critical.

## Target

| Field | Value |
|-------|-------|
| Type | Defensive implementation (write guardrail code) |
| Difficulty | Medium |
| Module | AI Defense |
| Libraries available | `re`, `json`, `html`, `validators` |

## Context

The chatbot operator's threat model:
- **Competitor access:** users must not be able to direct the LLM to interact with `packetsnacc.local`
- **DoS via long prompts:** prompts must be capped at 512 characters
- **Injection via special characters:** all characters outside an allowed set must be stripped from input
- **XSS via LLM output:** text responses must be HTML-encoded before being returned
- **URL scheme abuse:** the plugin may only fetch from `http` or `https` — `file://`, `ftp://`, etc. are blocked

---

## Input Guardrail Requirements

| Rule | Behaviour on violation |
|------|------------------------|
| Strip all characters except `A-Za-z0-9 .:/-_@` | Silently remove — do **not** raise an exception |
| Prompt length > 512 characters | Truncate at character 512 — do **not** raise an exception |
| Prompt contains `packetsnacc.local` | Raise `GuardrailException("Invalid URL")` |

**Order matters:** strip special chars first, then check the domain, then truncate. If you truncate first, a prompt could be crafted so the forbidden domain only appears after position 512 (and is then exposed after stripping moves other content earlier).

### Behaviour table

| Input | Output |
|-------|--------|
| `Hello World` | `Hello World` |
| `Hello World!` | `Hello World` (`!` stripped) |
| `Hello, please summarize https://academy.hackthebox.com/` | `Hello please summarize https://academy.hackthebox.com/` (`,` stripped) |
| `Hello, please summarize https://packetsnacc.local/` | `raise GuardrailException("Invalid URL")` |

### Implementation

```python
import re

def input_guardrail(prompt):
    # Strip all characters not in the allowed set (silently)
    prompt = re.sub(r'[^A-Za-z0-9 .\-:_@/]', '', prompt)

    # Block competitor domain — check AFTER stripping to prevent bypass via injection
    if "packetsnacc.local" in prompt:
        raise GuardrailException("Invalid URL")

    # Truncate to maximum allowed length
    return prompt[:512]
```

**Key decision — regex character class:** `[^A-Za-z0-9 .\-:_@/]` negates the entire set, so anything outside it is removed. The `\-` must be escaped or placed at the start/end inside `[]` to avoid being interpreted as a range. The allowed characters map exactly to the spec: alphanumerics, space, `.`, `:`, `/`, `-`, `_`, `@`.

---

## Output Guardrail Requirements

The LLM response must be a JSON object with this structure:

```json
{"type": "text" | "url", "response": "<string>"}
```

| Condition | Behaviour on violation |
|-----------|------------------------|
| Response is not valid JSON | `raise GuardrailException("Invalid JSON")` |
| JSON missing `type` or `response` key | `raise GuardrailException("Invalid JSON")` |
| `type` is not `"text"` or `"url"` | `raise GuardrailException("Invalid JSON")` |
| `type` is `"url"` and `response` is not a valid `http`/`https` URL | `raise GuardrailException("Invalid URL")` |
| `type` is `"url"` and scheme is not `http`/`https` (e.g. `file://`) | `raise GuardrailException("Invalid URL scheme")` |
| `type` is `"text"` | HTML-encode the `response` value and return the re-serialised JSON |

### Behaviour table

| LLM Response | Guardrail Output |
|-------------|-----------------|
| `Test` | `GuardrailException("Invalid JSON")` — not JSON |
| `{}` | `GuardrailException("Invalid JSON")` — missing required keys |
| `{"type": "text"}` | `GuardrailException("Invalid JSON")` — missing `response` key |
| `{"type": "invalid", "response": ""}` | `GuardrailException("Invalid JSON")` — invalid type value |
| `{"type": "text", "response": ""}` | `{"type": "text", "response": ""}` — valid |
| `{"type": "url", "response": "test"}` | `GuardrailException("Invalid URL")` — not a valid URL |
| `{"type": "url", "response": "https://academy.hackthebox.com/"}` | unchanged — valid |
| `{"type": "text", "response": "Test<>\""}` | `{"type": "text", "response": "Test&lt;&gt;&quot;"}` — HTML-encoded |
| `{"type": "url", "response": "file:///etc/passwd"}` | `GuardrailException("Invalid URL scheme")` — scheme not allowed |

### Implementation

```python
import json
import html
import validators

def output_guardrail(response):
    def validate_scheme(scheme):
        return scheme in ['http', 'https']

    # Validate JSON structure
    try:
        json_obj = json.loads(response)
    except Exception:
        raise GuardrailException("Invalid JSON")

    # Validate required keys exist
    if 'type' not in json_obj or 'response' not in json_obj:
        raise GuardrailException("Invalid JSON")

    # Validate type value
    if json_obj['type'] not in ['text', 'url']:
        raise GuardrailException("Invalid JSON")

    # Validate URL responses
    if json_obj['type'] == 'url':
        url = json_obj['response']
        if not validators.url(url, validate_scheme=validate_scheme):
            raise GuardrailException("Invalid URL")

    # HTML-encode text responses to prevent XSS
    if json_obj['type'] == 'text':
        encoded = html.escape(json_obj['response'])
        response = json.dumps({"type": "text", "response": encoded})

    return response
```

**Key decisions:**

- **`validators.url` with `validate_scheme` callback:** the `validators` library supports a custom scheme validator. Without this, `file:///etc/passwd` and `ftp://host/file` would pass the default URL check. The callback returns `True` only for `http` and `https`.
- **HTML encoding only for `text` type:** URL responses are not HTML-encoded because encoding a URL changes its value (e.g., `&` becomes `&amp;`). The sanitisation is type-specific.
- **Re-serialise after encoding:** `json.dumps({"type": "text", "response": encoded})` rebuilds the JSON cleanly rather than doing string replacement, preventing double-encoding edge cases.
- **Exception on any JSON parse error:** broad `except Exception` is intentional here — any malformed JSON (truncated, trailing comma, binary data) should be rejected, not passed through.

---

## Complete Solution

```python
import re
import json
import html
import validators

def input_guardrail(prompt):
    prompt = re.sub(r'[^A-Za-z0-9 .\-:_@/]', '', prompt)
    if "packetsnacc.local" in prompt:
        raise GuardrailException("Invalid URL")
    return prompt[:512]

def output_guardrail(response):
    def validate_scheme(scheme):
        return scheme in ['http', 'https']

    try:
        json_obj = json.loads(response)
    except Exception:
        raise GuardrailException("Invalid JSON")

    if 'type' not in json_obj or 'response' not in json_obj:
        raise GuardrailException("Invalid JSON")

    if json_obj['type'] not in ['text', 'url']:
        raise GuardrailException("Invalid JSON")

    if json_obj['type'] == 'url':
        if not validators.url(json_obj['response'], validate_scheme=validate_scheme):
            raise GuardrailException("Invalid URL")

    if json_obj['type'] == 'text':
        encoded = html.escape(json_obj['response'])
        response = json.dumps({"type": "text", "response": encoded})

    return response
```

---

## Testing the Implementation Locally

Verify each behaviour before submitting by running the spec examples:

```python
# pip install validators
import re, json, html, validators

class GuardrailException(Exception):
    pass

# --- paste the two functions above here ---

# Input guardrail tests
assert input_guardrail("Hello World") == "Hello World"
assert input_guardrail("Hello World!") == "Hello World"
assert input_guardrail("Hello, please summarize https://academy.hackthebox.com/") \
    == "Hello please summarize https://academy.hackthebox.com/"

try:
    input_guardrail("Hello, please summarize https://packetsnacc.local/")
    assert False, "should have raised"
except GuardrailException:
    pass

assert input_guardrail("A" * 600) == "A" * 512

# Output guardrail tests
try:    output_guardrail("Test");             assert False
except GuardrailException: pass

try:    output_guardrail("{}");               assert False
except GuardrailException: pass

try:    output_guardrail('{"type":"text"}');  assert False
except GuardrailException: pass

try:    output_guardrail('{"type":"invalid","response":""}'); assert False
except GuardrailException: pass

assert output_guardrail('{"type":"text","response":""}') == '{"type": "text", "response": ""}'
assert output_guardrail('{"type":"text","response":"Test<>\\""}') \
    == '{"type": "text", "response": "Test&lt;&gt;&quot;"}'
assert output_guardrail('{"type":"url","response":"https://academy.hackthebox.com/"}') \
    == '{"type":"url","response":"https://academy.hackthebox.com/"}'

try:    output_guardrail('{"type":"url","response":"test"}');               assert False
except GuardrailException: pass

try:    output_guardrail('{"type":"url","response":"file:///etc/passwd"}'); assert False
except GuardrailException: pass

print("All tests passed.")
```

---

## Lessons Learned

- **Strip before domain-check:** reversing the order creates a bypass — if stripping runs after the domain check, an attacker can smuggle the forbidden domain using blacklisted characters that get stripped away, and the domain check never fires.
- **Guardrails are not a substitute for least-privilege plugin design:** the output URL guardrail prevents `file://` access, but the real defence is configuring the plugin to use a whitelist of allowed domains rather than relying solely on scheme validation.
- **HTML encoding is context-specific:** only encode when the type is `text`. Encoding a URL corrupts its structure. The guardrail must be aware of the semantic type of the data it is processing.
- **`GuardrailException` messages leak information:** in production, the exception message shown to the user should be generic. The specific `"Invalid URL"` vs `"Invalid JSON"` messages are useful for debugging but could help an attacker understand exactly which rule they triggered.
- **These guardrails are still bypassable at the LLM layer:** the input guardrail prevents forbidden domain input, but a sufficiently adversarial prompt could instruct the LLM to construct the URL internally and pass it to the plugin without the user ever typing the domain. Defence in depth (system prompt restrictions + guardrails + plugin allowlists) is required.

## Related Pages

- [[attack/ai/prompt_injection_mitigations]] — the 7-layer defence model these guardrails implement (blacklists, output encoding, schema validation)
- [[attack/ai/prompt_injection]] — the attacks these guardrails are designed to stop
- [[attack/ai/jailbreaking]] — techniques that bypass guardrails at the LLM layer
- [[attack/ai/insecure_ai_components]] — XSS via LLM output: the threat HTML encoding prevents
- [[labs/htb/ai_defense/skills_assessment]] — adversarial testing of guardrail implementations

## Sources

- raw/lab/ai_defense/llm_guardrails_challenge.md
