---
tags: [enumeration, attack/web, tool, concept]
module: advanced_sql_injections
last_updated: 2026-05-22
source_count: 2
---

# Java Source Code — AI-Assisted SQLi Enumeration

Staged multi-agent pipeline that systematically enumerates SQL injection vulnerabilities in decompiled Java source. Accepts any JAR, WAR, or source directory. Three stages use pure Python; only the final checklist stage calls an LLM — and only for methods that passed all prior filters.

Tested against: BlueBird (Spring Boot / PostgreSQL).

---

## Why Staged — and What Open-Source Models Can and Cannot Do

The hypothesis that "narrow scope means small models work fine" is partially correct. Here is the precise boundary:

**What 7-13B models do reliably with narrow scope:**
- Answer binary YES/NO questions about visible syntax in a single method
- Return structured JSON when `format="json"` is set (85-90% reliability at 7B; ~95% at 13B)
- Identify `@RequestParam` and common transformation patterns (BCrypt, integer casts)

**What is NOT reliable regardless of model size:**
- **Cross-method variable tracing.** If a value is set in `methodA` and passed to `methodB` which builds SQL, a per-method agent cannot see the connection. This is a structural limitation of the approach. The pipeline flags these as `UNKNOWN` for manual follow-up.
- **Unrecognized transformation functions.** If the codebase uses `MyUtils.sanitize(email)`, the model does not know whether that function is safe. These are flagged.
- **Decompiler-renamed variables.** Fernflower produces `var10000`, `var1`, etc. Models misclassify these because provenance is lost. The assembler overrides any `INJECTABLE` verdict on variables matching `var\d+` to `UNKNOWN`.

**What does not need an LLM at all:**
- Stages 1 and 2 are pure Python. File sizing is arithmetic. Method boundary detection with regex and brace counting is more accurate and 100× faster than asking a model to parse decompiled output.

**Minimum recommended model:** `qwen2.5-coder:7b` or `codellama:13b`. A plain `codellama:7b` handles the checklist format but misses Spring JDBC idioms at a ~20% false-negative rate. `qwen2.5-coder:32b` (documented in [[tools/utility/ollama]]) is the best available option; use 7B only under VRAM constraints.

**Critical gotcha:** Ollama defaults to `num_ctx=2048` regardless of the model's actual capability. Every `ollama.chat()` call in this script passes `num_ctx` explicitly via `options={}`. Without this, large methods are silently truncated.

---

## Prerequisites

```bash
# Ollama running with a code model available
ollama pull qwen2.5-coder:7b   # or codellama:13b

# Python dependencies
pip install ollama requests

# Source files — decompile first if you have a JAR
# See: [[tools/enumeration/fernflower]]
# Or pass the JAR directly — the script extracts it automatically
```

---

## Pipeline Overview

```
JAR / WAR / source directory
        │
        ▼
[Python] Discover all .java files
        │
        ▼ FOR EACH FILE:
[Stage 1 — Python]   Size analysis
   Query Ollama /api/show for actual num_ctx
   Estimate tokens (chars ÷ 4)
   Compare to 50% of num_ctx → whole-file or chunked plan
        │
        ▼
[Stage 2 — Python]   Method inventory
   Regex + brace counting → method name, line range, has_sql flag
   Print inventory to console before any LLM call
        │
        ▼
[Stage 3 — Python]   Security filter
   HIGH:   SQL + concatenation + @RequestParam present
   MEDIUM: SQL + concatenation, no visible @RequestParam
   LOW:    SQL + parameterized patterns only → mark SAFE, skip LLM
        │
        ▼ HIGH + MEDIUM methods only:
[Stage 4 — Ollama]   Checklist analysis (one call per method)
   10-item YES/NO checklist → raw answers only, no verdict
        │
        ▼
[Python]  Assembler
   Deterministic verdict from checklist answers
   INJECTABLE / SAFE_VAR / UNKNOWN per variable
   VULNERABLE / NEEDS_REVIEW / SAFE / ERROR per method
        │
        ▼
   sqli_scan_report.json  +  console summary
```

The 50% context budget leaves room for the system prompt (~300 tokens), checklist items (~500 tokens), and the JSON response (~400 tokens). For a method-level call, most Java methods (20–80 lines) consume 400–1,600 tokens — well inside any 8k+ context.

---

## The Stage 4 Checklist

The model answers YES/NO per item and returns nothing else. The Python assembler derives all verdicts deterministically from the answers.

```
SQL QUERY STRUCTURE
1. Does the SQL string use + concatenation or String.format() to insert variable values?
2. Does the SQL string use ? placeholders bound via jdbcTemplate parameters or PreparedStatement?

FOR EACH VARIABLE CONCATENATED INTO SQL (answered per variable):
3. Is [var] declared @RequestParam, @PathVariable, or @RequestBody in the method signature?
4. Is [var] cast to int, long, Integer, or Long before reaching the SQL string?
5. Is [var] passed through BCrypt.hashpw(), MessageDigest, or a .encode() method?
6. Is [var] validated against a regex that restricts it to [A-Za-z0-9] characters only?
7. Is [var] sourced from the result of a prior parameterized jdbcTemplate query?
8. Is [var] a field of an object returned by a prior parameterized database call?

ERROR HANDLING (answered for the whole method):
9. Does a catch block pass exception messages directly to a response or model attribute?
10. Does the method check a client IP address before showing verbose error details?
```

**Assembler verdict logic:**
- `INJECTABLE`: item 3 = true AND items 4–8 all false
- `SAFE_VAR`: any of items 4–8 true
- `UNKNOWN`: item 3 = false AND items 4–8 all false (source unclear — manual trace)
- Variables matching `var\d+` are overridden to `UNKNOWN` regardless of other answers
- Method `VULNERABLE`: any variable `INJECTABLE`
- Method `NEEDS_REVIEW`: any variable `UNKNOWN`, none `INJECTABLE`
- Method `SAFE`: all variables `SAFE_VAR` or item 2 = true (parameterized)

---

## Implementation

Save as `java_sqli_scan.py`:

```python
#!/usr/bin/env python3
"""
java_sqli_scan.py — Staged multi-agent SQLi scanner for Java source code.
Accepts a .jar/.war file or a directory of .java files.

Stages:
  1 (Python)  Size analysis — token estimate vs model context window
  2 (Python)  Method inventory — regex + brace-count method boundaries
  3 (Python)  Security filter — prioritize HIGH/MEDIUM/LOW
  4 (Ollama)  Checklist analysis — YES/NO per item, assembler decides verdict
"""

import argparse
import json
import re
import shutil
import sys
import tempfile
import zipfile
from concurrent.futures import ThreadPoolExecutor, as_completed
from pathlib import Path

import ollama
import requests

# ---------------------------------------------------------------------------
# Patterns
# ---------------------------------------------------------------------------

SQL_KW      = re.compile(r"\b(SELECT|INSERT|UPDATE|DELETE|jdbcTemplate|JdbcTemplate)\b", re.I)
CONCAT_SQL  = re.compile(r'"\s*\+\s*\w|"\s*\+\s*\'|\'\s*\+\s*\w|String\.format\s*\(')
USER_PARAM  = re.compile(r'@(RequestParam|PathVariable|RequestBody)')
SAFE_JDBC   = re.compile(r'PreparedStatement|\.update\(sql,\s*(new\s+Object|\w+\[)|\.queryForObject\(sql,\s*(new\s+Object|\w+\[)')
METHOD_RE   = re.compile(r'^\s*(public|private|protected)(\s+static)?(\s+[\w<>\[\],\s]+)\s+(\w+)\s*\(')
DECOVAR_RE  = re.compile(r'^var\d+$')

CHECKLIST_PROMPT = """\
Analyze this Java method for SQL injection. Answer each item true or false only.

ITEMS:
1_concatenation   — SQL built with + concatenation or String.format()
2_parameterized   — SQL uses ? placeholders with bound parameters (PreparedStatement / jdbcTemplate Object[])
9_error_disclosure — catch block passes exception .getMessage() to a response attribute or HTTP response body
10_ip_gated_errors — method checks X-Forwarded-For or getRemoteAddr() before showing detailed errors

For each variable you identify as concatenated into the SQL string, report items 3-8:
3_request_param — variable is @RequestParam / @PathVariable / @RequestBody in this method's signature
4_int_cast      — variable is cast to int/long/Integer/Long before reaching the SQL string
5_hash          — variable passes through BCrypt.hashpw(), MessageDigest, or .encode() before SQL
6_regex_safe    — variable is validated against a regex that allows ONLY [A-Za-z0-9] characters
7_db_result     — variable comes from a prior parameterized jdbcTemplate query result
8_object_field  — variable is a field of an object from a prior parameterized database call

If a variable name looks like a decompiler artifact (var1, var10000) or its transformation
uses an unrecognized custom method, add a plain-English note to flags[].

Return ONLY valid JSON — no explanation, no markdown:
{{
  "1_concatenation": true|false,
  "2_parameterized": true|false,
  "9_error_disclosure": true|false,
  "10_ip_gated_errors": true|false,
  "variables": {{
    "<varname>": {{
      "3_request_param": true|false,
      "4_int_cast": true|false,
      "5_hash": true|false,
      "6_regex_safe": true|false,
      "7_db_result": true|false,
      "8_object_field": true|false
    }}
  }},
  "flags": []
}}

METHOD:
```java
{method_code}
```"""

# ---------------------------------------------------------------------------
# JAR / directory handling
# ---------------------------------------------------------------------------

def find_java_files(source: str) -> tuple[list[Path], Path | None]:
    """Accept a .jar/.war or directory. Returns (java_files, temp_dir_to_clean)."""
    p = Path(source)
    temp_dir = None

    if p.suffix in (".jar", ".war"):
        tmp = tempfile.mkdtemp(prefix="java_scan_")
        temp_dir = Path(tmp)
        with zipfile.ZipFile(p) as zf:
            zf.extractall(tmp)
        root = temp_dir
    elif p.is_dir():
        root = p
    else:
        print(f"[!] Not a JAR/WAR or directory: {source}")
        sys.exit(1)

    return sorted(root.rglob("*.java")), temp_dir

# ---------------------------------------------------------------------------
# Stage 1: Size analysis (Python only)
# ---------------------------------------------------------------------------

def get_model_context(model: str, base_url: str = "http://localhost:11434") -> int:
    """Query Ollama /api/show for the model's num_ctx. Defaults to 8192."""
    try:
        resp = requests.post(f"{base_url}/api/show", json={"model": model}, timeout=10)
        for line in resp.json().get("parameters", "").split("\n"):
            if "num_ctx" in line:
                parts = line.split()
                if parts and parts[-1].isdigit():
                    return int(parts[-1])
    except Exception:
        pass
    return 8192

def size_analysis(filepath: Path, model_ctx: int) -> dict:
    text = filepath.read_text(errors="replace")
    est_tokens = len(text) // 4
    budget = model_ctx // 2
    strategy = "whole_file" if est_tokens <= budget else "chunked"
    return {
        "lines": len(text.splitlines()),
        "estimated_tokens": est_tokens,
        "token_budget": budget,
        "strategy": strategy,
    }

# ---------------------------------------------------------------------------
# Stage 2: Method inventory (Python only)
# ---------------------------------------------------------------------------

def extract_methods(filepath: Path) -> list[dict]:
    """Regex + brace-counting method boundary detection. No LLM."""
    lines = filepath.read_text(errors="replace").splitlines()
    methods = []
    i = 0
    while i < len(lines):
        m = METHOD_RE.match(lines[i])
        if m:
            name = m.group(4)
            sig_line = i
            depth = 0
            body_start = None
            j = i
            found = False
            while j < len(lines):
                for ch in lines[j]:
                    if ch == "{":
                        depth += 1
                        if body_start is None:
                            body_start = j
                    elif ch == "}":
                        depth -= 1
                        if depth == 0 and body_start is not None:
                            body = "\n".join(lines[sig_line : j + 1])
                            methods.append({
                                "name": name,
                                "signature": lines[sig_line].strip(),
                                "start_line": sig_line + 1,
                                "end_line": j + 1,
                                "line_count": j - sig_line + 1,
                                "has_sql": bool(SQL_KW.search(body)),
                                "_body": body,
                            })
                            i = j + 1
                            found = True
                            break
                if found:
                    break
                j += 1
            if not found:
                i += 1
        else:
            i += 1
    return methods

# ---------------------------------------------------------------------------
# Stage 3: Security filter (Python only)
# ---------------------------------------------------------------------------

HIGH, MEDIUM, LOW = "HIGH", "MEDIUM", "LOW"

def classify_method(method: dict) -> str | None:
    """Return HIGH/MEDIUM/LOW priority or None if no SQL."""
    if not method["has_sql"]:
        return None
    body = method["_body"]
    has_concat  = bool(CONCAT_SQL.search(body))
    has_user    = bool(USER_PARAM.search(body))
    if not has_concat:
        return LOW
    return HIGH if has_user else MEDIUM

def filter_security_methods(methods: list[dict]) -> list[dict]:
    result = []
    for m in methods:
        pri = classify_method(m)
        if pri:
            result.append({**m, "priority": pri})
    return result

# ---------------------------------------------------------------------------
# Stage 4: Checklist analysis (Ollama — one call per method)
# ---------------------------------------------------------------------------

def run_checklist(method: dict, model: str, num_ctx: int) -> dict:
    prompt = CHECKLIST_PROMPT.format(method_code=method["_body"])
    try:
        resp = ollama.chat(
            model=model,
            messages=[{"role": "user", "content": prompt}],
            format="json",
            options={"num_ctx": num_ctx},
        )
        result = json.loads(resp["message"]["content"])
        result["parse_ok"] = True
        return result
    except json.JSONDecodeError as e:
        return {"parse_ok": False, "error": f"JSON parse error: {e}"}
    except Exception as e:
        return {"parse_ok": False, "error": str(e)}

# ---------------------------------------------------------------------------
# Assembler: deterministic verdict from checklist answers
# ---------------------------------------------------------------------------

def derive_verdict(checklist: dict) -> dict:
    if not checklist.get("parse_ok"):
        return {"method_verdict": "ERROR", "variables": {}, "injectable": [],
                "unknown": [], "flags": [checklist.get("error", "parse failed")]}

    variables = checklist.get("variables", {})
    var_verdicts, injectable, unknown = {}, [], []

    for var, ans in variables.items():
        safe = any([ans.get("4_int_cast"), ans.get("5_hash"),
                    ans.get("6_regex_safe"), ans.get("7_db_result"), ans.get("8_object_field")])
        user_src = ans.get("3_request_param", False)

        if DECOVAR_RE.match(var):
            verdict = "UNKNOWN"
            unknown.append(var)
        elif user_src and not safe:
            verdict = "INJECTABLE"
            injectable.append(var)
        elif safe:
            verdict = "SAFE_VAR"
        else:
            verdict = "UNKNOWN"
            unknown.append(var)
        var_verdicts[var] = verdict

    flags = list(checklist.get("flags", []))
    if unknown:
        flags.append(f"UNKNOWN source for: {unknown} — manual trace required")

    if injectable:
        method_verdict = "VULNERABLE"
    elif unknown:
        method_verdict = "NEEDS_REVIEW"
    elif checklist.get("2_parameterized") or not checklist.get("1_concatenation"):
        method_verdict = "SAFE"
    else:
        method_verdict = "NEEDS_REVIEW"

    return {
        "method_verdict": method_verdict,
        "variables": var_verdicts,
        "injectable": injectable,
        "unknown": unknown,
        "error_disclosure": checklist.get("9_error_disclosure", False),
        "ip_gated_errors": checklist.get("10_ip_gated_errors", False),
        "flags": flags,
    }

# ---------------------------------------------------------------------------
# Per-file orchestration
# ---------------------------------------------------------------------------

def analyze_file(filepath: Path, model: str, model_ctx: int) -> dict:
    result = {
        "file": filepath.name,
        "path": str(filepath),
        "size": size_analysis(filepath, model_ctx),
        "methods_analyzed": [],
    }
    methods     = extract_methods(filepath)
    sec_methods = filter_security_methods(methods)
    result["method_count"]     = len(methods)
    result["sql_method_count"] = len(sec_methods)

    for method in sec_methods:
        entry = {
            "method":     method["name"],
            "start_line": method["start_line"],
            "end_line":   method["end_line"],
            "priority":   method["priority"],
        }
        if method["priority"] in (HIGH, MEDIUM):
            checklist = run_checklist(method, model, model_ctx)
            entry["checklist"] = {k: v for k, v in checklist.items() if k != "parse_ok"}
            entry["verdict"]   = derive_verdict(checklist)
        else:
            entry["verdict"] = {"method_verdict": "SAFE",
                                 "note": "LOW priority — parameterized patterns only, skipped LLM"}
        result["methods_analyzed"].append(entry)

    return result

# ---------------------------------------------------------------------------
# Report assembly + output
# ---------------------------------------------------------------------------

def build_report(file_results: list[dict], model: str) -> dict:
    vulnerable, needs_review, safe = [], [], []
    for r in file_results:
        verdicts = [m.get("verdict", {}).get("method_verdict", "") for m in r.get("methods_analyzed", [])]
        if "VULNERABLE" in verdicts:
            r["file_verdict"] = "VULNERABLE"; vulnerable.append(r)
        elif "NEEDS_REVIEW" in verdicts or "ERROR" in verdicts:
            r["file_verdict"] = "NEEDS_REVIEW"; needs_review.append(r)
        else:
            r["file_verdict"] = "SAFE"; safe.append(r)
    return {
        "model": model,
        "totals": {
            "files_analyzed": len(file_results),
            "vulnerable":     len(vulnerable),
            "needs_review":   len(needs_review),
            "safe":           len(safe),
        },
        "vulnerability_map": vulnerable,
        "needs_review":      needs_review,
        "clean_files":       [r["file"] for r in safe],
    }

def print_inventory(fname: str, methods: list[dict]) -> None:
    print(f"\n  {fname}:")
    for m in methods:
        sql_tag = "[SQL]" if m["has_sql"] else "     "
        pri     = f" [{m['priority']}]" if m.get("priority") else ""
        print(f"    {sql_tag} L{m['start_line']:>4}-{m['end_line']:<4}  {m['name']}{pri}")

def print_summary(report: dict) -> None:
    t = report["totals"]
    print(f"\n{'='*60}")
    print(f"  Files analyzed  : {t['files_analyzed']}")
    print(f"  Vulnerable      : {t['vulnerable']}")
    print(f"  Needs review    : {t['needs_review']}")
    print(f"  Clean           : {t['safe']}")
    print(f"{'='*60}")
    for r in report.get("vulnerability_map", []):
        print(f"\n[VULNERABLE] {r['file']}")
        for m in r.get("methods_analyzed", []):
            v = m.get("verdict", {})
            if v.get("method_verdict") == "VULNERABLE":
                print(f"  {m['method']} (L{m['start_line']}-{m['end_line']})")
                print(f"    Injectable : {v.get('injectable', [])}")
                safe_v = [k for k, s in v.get("variables", {}).items() if s == "SAFE_VAR"]
                print(f"    Safe vars  : {safe_v}")
                if v.get("flags"):
                    print(f"    Flags      : {v['flags']}")
                if v.get("error_disclosure"):
                    print(f"    [!] Exception message exposed in response")
                if v.get("ip_gated_errors"):
                    print(f"    [!] IP-gated verbose errors — X-Forwarded-For bypass likely")
    for r in report.get("needs_review", []):
        print(f"\n[NEEDS REVIEW] {r['file']}")
        for m in r.get("methods_analyzed", []):
            v = m.get("verdict", {})
            if v.get("method_verdict") in ("NEEDS_REVIEW", "ERROR"):
                print(f"  {m['method']}: {v.get('flags', ['manual trace required'])}")

# ---------------------------------------------------------------------------
# Entry point
# ---------------------------------------------------------------------------

def main():
    ap = argparse.ArgumentParser(
        description="Staged Java SQLi scanner — works with any JAR, WAR, or source directory."
    )
    ap.add_argument("source",    help=".jar/.war file or directory containing .java files")
    ap.add_argument("--model",   default="qwen2.5-coder:7b")
    ap.add_argument("--workers", type=int, default=2, help="Parallel Stage 4 workers")
    ap.add_argument("--out",     default="sqli_scan_report.json")
    args = ap.parse_args()

    print(f"[*] Source: {args.source}")
    java_files, temp_dir = find_java_files(args.source)
    print(f"[*] Found {len(java_files)} .java files")

    print(f"\n[*] Querying model context window ({args.model})...")
    model_ctx = get_model_context(args.model)
    print(f"[*] num_ctx={model_ctx}  |  token budget per call: {model_ctx//2}")

    # Stages 1-3: fast, sequential, print inventory before LLM
    print(f"\n[*] Stage 1-3: Size check, method inventory, security filter")
    sql_files = []
    for f in java_files:
        methods     = extract_methods(f)
        sec_methods = filter_security_methods(methods)
        if sec_methods:
            sql_files.append(f)
            print_inventory(f.name, sec_methods)

    high_med_count = sum(
        1 for f in sql_files
        for m in filter_security_methods(extract_methods(f))
        if m["priority"] in (HIGH, MEDIUM)
    )
    print(f"\n[*] {high_med_count} HIGH/MEDIUM methods queued for Stage 4 LLM analysis")
    if not sql_files:
        print("[*] No SQL-touching files found. Exiting.")
        return

    # Stage 4: parallel LLM checklist
    print(f"\n[*] Stage 4: Checklist analysis ({args.workers} parallel workers)")
    results = []
    with ThreadPoolExecutor(max_workers=args.workers) as pool:
        futures = {pool.submit(analyze_file, f, args.model, model_ctx): f for f in sql_files}
        done = 0
        for future in as_completed(futures):
            done += 1
            result = future.result()
            results.append(result)
            verdicts = [m.get("verdict", {}).get("method_verdict", "") for m in result.get("methods_analyzed", [])]
            tag = "VULNERABLE" if "VULNERABLE" in verdicts else ("NEEDS_REVIEW" if "NEEDS_REVIEW" in verdicts else "clean")
            print(f"  [{done}/{len(sql_files)}] {result['file']} -> {tag}")

    if temp_dir:
        shutil.rmtree(temp_dir, ignore_errors=True)

    report = build_report(results, args.model)
    Path(args.out).write_text(json.dumps(report, indent=2))
    print(f"\n[+] Report written to {args.out}")
    print_summary(report)

if __name__ == "__main__":
    main()
```

---

## Running the Scan

```bash
# Against a JAR (script extracts to temp dir automatically)
python3 java_sqli_scan.py target.jar

# Against a decompiled source directory
python3 java_sqli_scan.py BlueBirdSourceCode/

# Specify model and workers
python3 java_sqli_scan.py target.jar --model codellama:13b --workers 4

# Custom output
python3 java_sqli_scan.py target.jar --out engagement_sqli.json
```

---

## Sample Output

**Stage 1-3 inventory (printed before any LLM call):**
```
[*] Stage 1-3: Size check, method inventory, security filter

  AuthController.java:
    [SQL] L  78-120   forgotPOST [HIGH]
    [SQL] L 145-185   signupPOST [HIGH]
    [SQL] L  45-77    resetPOST  [LOW]

  IndexController.java:
    [SQL] L  60-95    findUser   [HIGH]

  ProfileController.java:
    [SQL] L  28-55    profile    [MEDIUM]

[*] 4 HIGH/MEDIUM methods queued for Stage 4 LLM analysis
```

**Stage 4 console summary:**
```
[*] Stage 4: Checklist analysis (2 parallel workers)
  [1/3] AuthController.java -> VULNERABLE
  [2/3] IndexController.java -> VULNERABLE
  [3/3] ProfileController.java -> NEEDS_REVIEW

============================================================
  Files analyzed  : 3
  Vulnerable      : 2
  Needs review    : 1
  Clean           : 0
============================================================

[VULNERABLE] AuthController.java
  signupPOST (L145-185)
    Injectable : ['name', 'username', 'email']
    Safe vars  : ['passwordHash']
  forgotPOST (L78-120)
    Injectable : ['email']
    Safe vars  : []
    [!] IP-gated verbose errors — X-Forwarded-For bypass likely

[VULNERABLE] IndexController.java
  findUser (L60-95)
    Injectable : ['u']
    Safe vars  : []

[NEEDS REVIEW] ProfileController.java
  profile: ['UNKNOWN source for: [user.getEmail()] — manual trace required']
```

The `ProfileController.java` NEEDS_REVIEW result is the correct outcome: `user.getEmail()` is sourced from an earlier parameterized query, but the per-method agent cannot see that. Apply [[attack/web/second_order_sqli]] methodology to trace whether the email column is writable via another injectable endpoint.

---

## Interpreting the JSON Report

| Field | Meaning |
|---|---|
| `file_verdict: VULNERABLE` | At least one method has a confirmed `INJECTABLE` variable |
| `file_verdict: NEEDS_REVIEW` | At least one `UNKNOWN` variable — cross-method trace needed |
| `variable verdict: INJECTABLE` | `@RequestParam`/`@PathVariable`/`@RequestBody` with no safe transformation |
| `variable verdict: SAFE_VAR` | Confirmed safe: BCrypt hash, integer cast, safe-charset regex, or DB-sourced |
| `variable verdict: UNKNOWN` | Source unclear or decompiler artifact — manual read required |
| `error_disclosure: true` | Exception `.getMessage()` exposed to client — error-based SQLi may be possible |
| `ip_gated_errors: true` | X-Forwarded-For bypass: set header to `127.0.1.1` to unlock stack traces |
| `flags` | Model uncertainty notes — any flagged method needs manual verification |

---

## What This Pipeline Cannot Detect

- **Cross-method data flow.** A value set in `setEmail()` and consumed in `getEmail()` inside another SQL method is invisible to per-method agents. `NEEDS_REVIEW` is the correct output; manual trace via [[attack/web/white_box_sqli_methodology]] is required.
- **Custom sanitization functions.** `MyUtils.escape(input)` will be flagged as UNKNOWN unless the model recognizes it. Add known-safe functions to the prompt if the target codebase uses them.
- **Non-JDBC SQL access.** Hibernate JPQL, JPA criteria API, and MyBatis mapper XML are not covered by the current grep patterns. Extend `SQL_KW` if the target uses these ORMs.
- **Second-order injection.** Confirmed SAFE values stored in the DB and later used in a concatenated query. Identified as NEEDS_REVIEW when `7_db_result=true` — see [[attack/web/second_order_sqli]].

---

## Gotchas & Notes

- Run `ollama list` before scanning — a cold model pull during analysis blocks the first worker for several minutes
- `--workers 1` on machines with <8 GB VRAM; parallel contexts OOM without warning on small GPUs
- Fernflower-decompiled code uses `var10000`-style names for some locals; the assembler automatically overrides these to `UNKNOWN` — check the flags list in the report
- `qwen2.5-coder` models follow the strict JSON checklist format more reliably than `codellama` at the same parameter count; prefer them when available

## Related Pages

- [[tools/utility/ollama]] — model management, VRAM sizing, context window configuration
- [[tools/enumeration/fernflower]] — decompile JARs before scanning (or pass JAR directly)
- [[attack/web/white_box_sqli_methodology]] — manual grep-based approach; use alongside this scanner for cross-method traces
- [[labs/htb/advanced_sql_injections/searching_for_strings]] — the variable-tracing methodology encoded in the Stage 4 checklist
- [[attack/web/second_order_sqli]] — follow-up for NEEDS_REVIEW findings where DB-sourced values feed into SQL

## Sources

- raw/advanced_sql_injections/identifying_vulnerabilities_searching_for_strings.md
- raw/advanced_sql_injections/identifying_vulnerabilities_decompiling_java_archives.md
