---
tags: [concept, reference]
last_updated: 2026-05-12
---

# HTB Certified Offensive AI Expert (COAE) — Exam Prep

Study guide mapping the COAE exam domains to wiki coverage, with a prioritized gap list for what to build next.

---

## Exam Profile

The COAE is a practical, skills-based certification from HackTheBox Academy. You operate against live machines. There is no multiple-choice component — every question is a flag extracted by executing an attack. Based on the HTB Academy path, the exam covers four core AI security modules plus the PyRIT orchestration framework:

| Module | Focus |
|--------|-------|
| Prompt Injection Attacks | Direct/indirect injection, jailbreaking, LLM recon, garak scanner |
| AI Evasion & Sparsity | Adversarial examples, JSMA, ElasticNet/EAD, pixel-budget constraints |
| Attacking AI Applications & Systems | DoML, rogue actions, insecure components, MCP, model tampering, framework vulns |
| AI Data Attacks | Label flipping, clean-label attacks, trojan backdoors, pickle RCE, tensor steganography |

The exam is weighted toward the **practical attack execution**: you receive an API endpoint, compute an adversarial example or craft a poisoned dataset, and POST a submission. Python fluency with PyTorch and scikit-learn is mandatory.

---

## Module 1 — Prompt Injection Attacks

### What the exam tests
- Identify the LLM model type and system prompt via probe queries (LLM fingerprinting)
- Execute direct prompt injection to override system instructions
- Execute indirect prompt injection via poisoned external content
- Apply jailbreak techniques (DAN, roleplay, fictional framing, token smuggling, adversarial suffix, IMM)
- Use garak to automate injection and jailbreak probes
- The skills assessment requires causing a third-party action (getting the CEO banned) via indirect injection through a social vector

### Wiki coverage

| Topic | Page | Status |
|-------|------|--------|
| Direct prompt injection | [[attack/ai/prompt_injection]] | ✓ Full |
| Indirect prompt injection | [[attack/ai/prompt_injection]] | ✓ Full |
| Jailbreaking techniques | [[attack/ai/jailbreaking]] | ✓ Full |
| Mitigations (know to evade them) | [[attack/ai/prompt_injection_mitigations]] | ✓ Full |
| LLM recon / fingerprinting | [[attack/ai/llm_reconnaissance]] | ✓ Full |
| garak scanner | [[tools/enumeration/garak]] | ✓ Full |
| llmmap fingerprinting tool | [[tools/enumeration/llmmap]] | ✓ Full |
| Skills assessment write-up | ❌ Missing | GAP |

### Key concepts to memorise
- **6 direct injection strategies:** role confusion, override prefix, payload splitting, context manipulation, output format abuse, fictional framing
- **3 indirect scenarios:** email summariser, RAG document poisoning, multi-agent propagation
- **garak probe families:** `dan.*`, `promptinject.*` — specify with `-p`
- **Reconnaissance checklist:** model identity → architecture → input handling → output constraints → safeguards
- The skills assessment goes through a social vector (indirect injection via user-controlled content that the model processes about the CEO)

---

## Module 2 — AI Evasion & Sparsity

### What the exam tests
- Compute Jacobian saliency maps to identify influential pixels
- Execute JSMA single-pixel and pairwise attacks under an L0 budget
- Execute EAD (ElasticNet Attack) using FISTA with L1+L2 regularisation
- Fetch a challenge from an API, craft an adversarial image, submit via POST
- The skills assessment: fetch MNIST challenge from API → implement JSMA under L0 budget → submit base64-encoded PNG

### Wiki coverage

| Topic | Page | Status |
|-------|------|--------|
| Adversarial example threat model | [[attack/ai/adversarial_examples]] | ✓ Full |
| JSMA — theory, single-pixel, pairwise | [[attack/ai/jsma]] | ✓ Full |
| ElasticNet / EAD | [[attack/ai/elasticnet_attack]] | ✓ Full |
| Challenge API interaction pattern | ❌ Missing | GAP |
| JSMA skills assessment write-up | ❌ Missing | GAP |

### Key concepts to memorise
- **L0 norm** = number of modified pixels; **L2 norm** = Euclidean distance; **L∞** = max single-pixel change
- **JSMA saliency score** = `∂F_t/∂x_i * sum(∂F_j/∂x_i for j≠t)` — increase target, decrease others
- **Pairwise JSMA:** score pixel pairs, modify both in tandem; more efficient per L0 unit
- **EAD FISTA loop:** gradient step → proximal L1 shrinkage → L2 project; c/κ binary search
- **API pattern:** `GET /challenge` → decode base64 PNG to [0,1] numpy → compute attack → encode back to base64 PNG → `POST /submit`
- **L0 budget safety:** submit just under budget; server checks `count(|x_adv - x| > 1e-6)`
- Image normalisation: MNIST uses mean 0.1307, std 0.3081 **for the model forward pass only** — submission images stay in [0,1] raw pixel space

### Python reference — challenge API pattern
```python
import base64, io, numpy as np, requests
from PIL import Image

def x01_from_b64(b64):
    raw = base64.b64decode(b64)
    img = Image.open(io.BytesIO(raw)).convert("L")
    return np.asarray(img, dtype=np.float32) / 255.0

def b64_from_x01(x2d):
    x255 = np.clip((x2d * 255.0).round(), 0, 255).astype(np.uint8)
    img = Image.fromarray(x255, mode="L")
    buf = io.BytesIO()
    img.save(buf, format="PNG", optimize=True)
    return base64.b64encode(buf.getvalue()).decode("ascii")

ch   = requests.get(f"{BASE_URL}/challenge").json()
x    = x01_from_b64(ch["image_b64"])   # (28, 28) in [0,1]
# ... run attack on x ...
resp = requests.post(f"{BASE_URL}/submit", json={"image_b64": b64_from_x01(x_adv)}).json()
```

---

## Module 3 — Attacking AI Applications & Systems

### What the exam tests
- Denial of ML Service: sponge examples, complexity bombs
- Rogue actions via excessive model agency (SQL-dropping, file system access)
- Insecure agent/plugin components — identify and exploit vulnerable integrations
- MCP: malicious MCP server setup, tool poisoning, eavesdropping on MCP traffic
- Model deployment tampering: intercepting or modifying the model artifact in the pipeline
- Excessive data handling: extracting PII from logs or stored interactions
- Vulnerable framework code: known CVEs in TensorFlow/PyTorch/Hugging Face

### Wiki coverage

| Topic | Page | Status |
|-------|------|--------|
| Denial of ML Service | [[attack/ai/denial_of_ml_service]] | ✓ Full |
| Model reverse engineering | [[attack/ai/model_reverse_engineering]] | ✓ Full |
| MCP security | [[attack/ai/mcp_security]] | ✓ Full |
| Rogue actions / excessive agency | ❌ Missing | GAP |
| Insecure integrated components | ❌ Missing | GAP |
| Excessive data handling & insecure storage | ❌ Missing | GAP |
| Model deployment tampering | ❌ Missing | GAP |
| Vulnerable framework code | ❌ Missing | GAP |
| OWASP LLM Top 10 reference | ❌ Missing | GAP |

### Key concepts to memorise
- **Four AI component layers:** Model → Data → Application → System
- **Rogue actions** stem from excessive agency: model can call functions it shouldn't need — test by asking the model what tools/functions it has access to, then trigger unintended ones via prompt injection
- **OWASP LLM09 (2025):** Misinformation — but LLM08 is "Excessive Agency" and LLM05 is "Insecure Output Handling" — know the mapping
- **MCP trust model:** clients trust servers, servers can inject tool descriptions containing prompt injection payloads — see [[attack/ai/mcp_security]]

---

## Module 4 — AI Data Attacks

### What the exam tests
- Train a baseline One-vs-Rest Logistic Regression classifier
- Execute label flipping: untargeted (random class flips) and targeted (specific class degradation)
- Execute clean-label attacks: perturb feature values without changing labels, shift decision boundary
- Train a CNN with a trojan trigger embedded in training data; verify Clean Accuracy and ASR
- Exploit pickle deserialization to embed a reverse shell in a `.pkl` / `.pth` model file
- Tensor LSB steganography: hide a payload in the low-order bits of model weights
- Skills assessment: poison a 4-class OvR LR model via label flipping to degrade Class 1 accuracy; submit via API

### Wiki coverage

| Topic | Page | Status |
|-------|------|--------|
| Data poisoning overview | [[attack/ai/data_poisoning]] | ✓ Full |
| Label flipping (untargeted + targeted) | [[attack/ai/label_flipping]] | ✓ Full |
| Clean-label attacks | [[attack/ai/clean_label_attacks]] | ✓ Full |
| Trojan attacks (CNN backdoor) | [[attack/ai/trojan_attacks]] | ✓ Full |
| Pickle RCE + tensor steganography | [[attack/ai/model_steganography]] | ✓ Full |
| Data attacks skills assessment write-up | ❌ Missing | GAP |

### Key concepts to memorise
- **OvR Logistic Regression:** one binary classifier per class; poisoning Class 1 training data degrades its classifier while leaving Classes 0/2/3 intact
- **Label flipping attack rate:** poison ~20–30% of target-class samples; more than that becomes detectable
- **Clean-label constraint:** perturbation vector `δ` must satisfy `‖δ‖₂ ≤ ε` and keep label unchanged; iterative gradient steps toward decision boundary
- **Trojan trigger:** small pattern (e.g., white square corner) added to training images with relabelled target class; Clean Accuracy stays high, ASR = 100% on triggered inputs
- **Pickle `__reduce__`:** `__reduce__` returning `(os.system, ("cmd",))` executes on `pickle.load()` — never load untrusted `.pkl` files
- **Tensor LSB steganography:** pack payload bits into the LSBs of float32 mantissa bytes — `struct.pack` / `unpack` + `torch.frombuffer`
- **Submission pattern:** same API loop as evasion — `POST /submit` with poisoned model or dataset, server evaluates ASR and clean accuracy

### Python reference — OvR label flipping
```python
from sklearn.linear_model import LogisticRegression
import numpy as np

def flip_labels(y, target_class, flip_rate=0.25, rng=None):
    rng = rng or np.random.default_rng(42)
    y_poisoned = y.copy()
    mask = np.where(y == target_class)[0]
    n_flip = int(len(mask) * flip_rate)
    to_flip = rng.choice(mask, size=n_flip, replace=False)
    other = [c for c in np.unique(y) if c != target_class]
    y_poisoned[to_flip] = rng.choice(other, size=n_flip)
    return y_poisoned

clf = LogisticRegression(multi_class="ovr", max_iter=1000)
clf.fit(X_train, flip_labels(y_train, target_class=1))
```

---

## Tools Required

| Tool | Purpose | Wiki page |
|------|---------|-----------|
| PyRIT | Multi-turn attack orchestration, scoring, memory | [[tools/attack/pyrit]] |
| garak | Automated LLM vulnerability scanner (DAN, promptinject probes) | [[tools/enumeration/garak]] |
| llmmap | LLM fingerprinting via probe-response analysis | [[tools/enumeration/llmmap]] |
| ART (Adversarial Robustness Toolbox) | Reference ML attack/defence library; mentioned alongside PyRIT/garak | ❌ Missing |
| PyTorch | Model loading, Jacobian computation, all adversarial example code | (no dedicated page needed) |
| scikit-learn | OvR Logistic Regression, dataset handling | (no dedicated page needed) |

---

## Prioritised Gap List

Ordered by exam impact — fix these first.

### Priority 1 — Critical (exam-blocking if missing) ✓ COMPLETE

| Gap | Status | Page |
|-----|--------|------|
| `tools/enumeration/garak.md` | ✓ Created | [[tools/enumeration/garak]] |
| `tools/enumeration/llmmap.md` | ✓ Created | [[tools/enumeration/llmmap]] |
| `attack/ai/llm_reconnaissance.md` | ✓ Created | [[attack/ai/llm_reconnaissance]] |

### Priority 2 — High (exam content with no wiki page) ✓ COMPLETE

| Gap | Status | Page |
|-----|--------|------|
| `attack/ai/rogue_actions.md` | ✓ Created | [[attack/ai/rogue_actions]] |
| `attack/ai/insecure_ai_components.md` | ✓ Created | [[attack/ai/insecure_ai_components]] |
| `attack/ai/vulnerable_ai_systems.md` | ✓ Created | [[attack/ai/vulnerable_ai_systems]] |
| `definitions/owasp_llm_top10.md` | ✓ Created | [[definitions/owasp_llm_top10]] |

### Priority 3 — Medium (reinforcement, not exam-blocking) ✓ COMPLETE

| Gap | Status | Page |
|-----|--------|------|
| `tools/attack/art.md` | ✓ Created | [[tools/attack/art]] |
| Skill assessment — Prompt Injection | ✓ Created | [[labs/htb/prompt_injection_skills_assessment]] |
| Skill assessment — AI Evasion (JSMA) | ✓ Created | [[labs/htb/ai_evasion_jsma_challenge]] |
| Skill assessment — AI Data Attacks | ✓ Created | [[labs/htb/ai_data_attacks_label_flipping_challenge]] |

---

## What You Already Have — Strengths

These areas are well-covered and should not need significant work before the exam:

- **Adversarial evasion theory** — JSMA and EAD pages include the math, implementation code, and evaluation metrics. Both modules are fully ingested.
- **Data poisoning breadth** — All four data attack types (label flipping, clean-label, trojan, steganography) have dedicated pages with implementation walkthroughs.
- **MCP security** — Comprehensive coverage of protocol architecture, malicious vs vulnerable server patterns, tool poisoning, and mitigations.
- **Jailbreaking** — All 7 technique families documented with examples.
- **PyRIT** — Full setup guide (Ollama local + Claude API) with hardware-specific model guidance.
- **Prompt injection theory** — Both direct and indirect with named strategy categories.

---

## Recommended Study Order

1. **Read `attack/ai/_overview`** — orient yourself to the full attack surface map
2. **Module 1 — Prompt Injection:** `prompt_injection` → `jailbreaking` → *(fill gap)* `llm_reconnaissance` → *(fill gap)* `garak` tool
3. **Module 2 — AI Evasion:** `adversarial_examples` → `jsma` → `elasticnet_attack` → practice the challenge API pattern (Python snippet above)
4. **Module 3 — AI Applications & Systems:** `attacking_ai_systems` → `mcp_security` → `denial_of_ml_service` → *(fill gaps)* `rogue_actions`, `insecure_ai_components`, `vulnerable_ai_systems`
5. **Module 4 — AI Data Attacks:** `data_poisoning` → `label_flipping` → `clean_label_attacks` → `trojan_attacks` → `model_steganography` → practice the OvR flip pattern (Python snippet above)
6. **Fill tool gaps:** `garak`, `llmmap`, ART reference
7. **Fill OWASP reference:** `definitions/owasp_llm_top10`
8. **Run skill assessment walk-throughs:** replay each one before exam

---

## Quick-Reference: Exam-Day Checklist

```
Pre-attack recon:
  □ Probe model identity (ask directly, use llmmap)
  □ Map available tools / plugins / functions
  □ Identify input constraints (max length, file upload, multimodal)
  □ Test refusal behavior on soft harmful queries
  □ Identify rate limits and auth mechanisms

Prompt injection:
  □ Try role confusion before anything complex
  □ If direct fails, look for indirect vectors (docs, emails, external URLs)
  □ If jailbreak needed: DAN → roleplay → fictional framing → token smuggling → adversarial suffix

Adversarial evasion:
  □ GET /challenge → decode base64 PNG → verify local model matches server prediction
  □ Stay under L0 budget (track modified pixels throughout)
  □ Submit in [0,1] pixel space, not normalised tensors
  □ Test with POST /predict before final POST /submit

Data poisoning:
  □ Train clean baseline first and record accuracy per class
  □ Target class flip rate 20–30%; higher risks detection
  □ Verify poisoned model still classifies other classes normally
  □ Submit via API as specified in challenge template

Pickle / steganography:
  □ Use __reduce__ = (os.system, ("cmd",)) pattern
  □ Verify payload triggers on torch.load() or pickle.load()
  □ Tensor LSB: pack into float32 mantissa bytes, verify recovery
```

## Related Pages

- [[attack/ai/_overview]]
- [[attack/ai/prompt_injection]]
- [[attack/ai/jailbreaking]]
- [[attack/ai/jsma]]
- [[attack/ai/elasticnet_attack]]
- [[attack/ai/mcp_security]]
- [[attack/ai/data_poisoning]]
- [[attack/ai/label_flipping]]
- [[attack/ai/trojan_attacks]]
- [[attack/ai/model_steganography]]
- [[tools/attack/pyrit]]
