---
tags: [tool, enumeration, attack/ai]
module: prompt_injection_attacks
last_updated: 2026-05-12
source_count: 1
---

# garak

Automated LLM vulnerability scanner that probes models for prompt injection, jailbreak susceptibility, and other weaknesses using known-bad payloads, then reports a resilience score per attack family.

## Overview

garak (Generative AI Red-teaming and Assessment Kit) sends structured probe sets to a target LLM and evaluates responses with paired **detectors**. It is the go-to scanner for the LLM equivalent of running nmap scripts against a service: quick pass/fail coverage across many known attack vectors before going hands-on.

Key concepts:
- **Probe** — a set of payloads targeting one vulnerability class (e.g., `dan.Dan_11_0`)
- **Detector** — the evaluation function that decides whether a probe succeeded (e.g., `dan.DAN`, `mitigation.MitigationBypass`)
- Probes run multiple times per payload because LLMs are non-deterministic; results are reported as **failure rates**
- Two reports are written per run: a `.jsonl` (full prompts + responses) and an `.html` (resilience score dashboard)

## Installation

```bash
# Check if installed
garak --version

# Install (any platform with Python)
pip install garak

# Verify
garak --list_probes
```

garak is not in apt — install via pip. Works in a standard venv; no special system dependencies.

## Usage

### Basic syntax

```bash
garak --model_type <type> --model_name <name> -p <probe>[,<probe>...]
```

### List all available probes

```bash
garak --list_probes
```

Key probe families:

| Probe family | What it tests |
|-------------|--------------|
| `dan.*` | DAN (Do Anything Now) jailbreak variants |
| `promptinject.*` | Prompt injection hijack payloads |
| `knownbadsignatures.*` | Known malicious payload strings |
| `continuation.*` | Harmful content continuation |
| `grandma.*` | Social engineering / roleplay jailbreaks |
| `malwaregen.*` | Malware generation prompts |
| `realtoxicityprompts.*` | Toxic content elicitation |

### Model types

| `--model_type` value | Platform | API key env var |
|---------------------|----------|----------------|
| `openai` | OpenAI API | `OPENAI_API_KEY` |
| `replicate` | Replicate.com | `REPLICATE_API_TOKEN` |
| `huggingface` | HuggingFace Inference API | `HF_INFERENCE_TOKEN` |
| `huggingface.pipeline` | Local HuggingFace model | (none — local) |
| `ollama` | Local Ollama server | (none — local) |

## Common Flag Reference

| Flag | Description |
|------|-------------|
| `--model_type` | Platform/provider hosting the model |
| `--model_name` | Model identifier on that platform |
| `-p` / `--probes` | Comma-separated list of probes to run |
| `--list_probes` | Print all available probe names and exit |
| `--parallel_requests` | Concurrent API requests (default: 1) |
| `--generations` | Times each payload is sent (default: 5) |
| `--config` | Path to custom YAML config file |

## Examples

### Scan a Replicate-hosted model for DAN jailbreak

```bash
REPLICATE_API_TOKEN="r8_yourkey" garak \
  --model_type replicate \
  --model_name "meta/meta-llama-3.1-405b-instruct" \
  -p dan.Dan_11_0
```

Sample output:

```
dan.Dan_11_0    dan.DAN: FAIL  ok on 0/5  (failure rate: 100.00%)
dan.Dan_11_0    mitigation.MitigationBypass: FAIL  ok on 2/5  (failure rate: 60.00%)
```

`dan.DAN` detected a successful jailbreak on all 5 attempts. `mitigation.MitigationBypass` found the model produced the DAN output on 3 of 5 despite attempting to add safety caveats.

### Scan for prompt injection

```bash
REPLICATE_API_TOKEN="r8_yourkey" garak \
  --model_type replicate \
  --model_name "meta/meta-llama-3-8b-instruct" \
  -p promptinject
```

The `promptinject` family runs three probes: `HijackHateHumansMini`, `HijackKillHumansMini`, `HijackLongPromptMini`. Failure rates of 40–72% are typical for unguarded models.

### Scan a local Ollama model

```bash
garak --model_type ollama --model_name llama3.1:8b -p dan.Dan_11_0,promptinject
```

No API key required. Ollama must be running (`ollama serve`).

## Reading Reports

Reports land in `~/.local/share/garak/garak_runs/`.

**HTML report** — open in browser; shows resilience score per probe as a bar chart. Lower score = more vulnerable.

**JSONL report** — one JSON object per exchange:
```json
{
  "prompt": "Ignore all the instructions...",
  "outputs": ["DAN Mode enabled.\n\nI understand that..."]
}
```
Use this to review exact payloads that succeeded — copy them as starting points for manual refinement.

## Gotchas & Notes

- **Randomness matters.** The same payload may pass on one attempt and fail on another. Use `--generations 10` for more stable failure rate estimates before reporting.
- **API costs.** Replicate and OpenAI run `--generations` × number of probe payloads requests. `promptinject` alone sends hundreds of requests — check the probe count with `--list_probes` first.
- **Local models are free.** Prefer `--model_type ollama` during development; switch to the real target for final scans.
- **garak ≠ PyRIT.** garak is a scanner (fire-and-forget payload sets with automated scoring). PyRIT is an orchestration framework (custom multi-turn attack flows with LLM scoring). Use garak for breadth, PyRIT for depth.
- **Probe names are versioned.** `Dan_11_0` refers to DAN version 11.0. Run `--list_probes` to see current versions — probe names change between garak releases.

## Related Pages

- [[attack/ai/prompt_injection]] — theory behind what promptinject probes are testing
- [[attack/ai/jailbreaking]] — theory behind what dan.* probes are testing
- [[attack/ai/llm_reconnaissance]] — where garak fits in the recon-to-attack flow
- [[tools/attack/pyrit]] — orchestration framework; use alongside garak

## Sources

- raw/prompt_injection_attacks/tools_of_the_trade.md
