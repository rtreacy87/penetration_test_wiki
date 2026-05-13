---
tags: [tool, attack/ai, concept]
module: pyrit
last_updated: 2026-05-12
source_count: 4
---

# PyRIT

Microsoft's open-source Python Risk Identification Toolkit for AI red-teaming — orchestrates multi-turn attacks against LLMs (prompt injection, jailbreaking, harmful content extraction) and scores responses automatically.

## Overview

PyRIT is the framework used by Microsoft's AI Red Team (AIRT). It handles the plumbing of AI red-teaming:

- Sends attack prompts (single-turn or multi-turn) to a **target** model
- Optionally passes responses through a **scorer** (another LLM or rule-based judge) to evaluate success
- Logs every prompt/response pair to a **memory database** (SQLite or Azure SQL)
- Supports **converters** that transform prompts (encoding, translation, paraphrasing) before sending

The typical PyRIT attack loop: orchestrator → converter → target → scorer → memory.

## Installation

```bash
# Prerequisites: Python 3.12, git, uv, Node.js ≥18

# Install uv (Linux/macOS)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Clone and install
git clone https://github.com/microsoft/PyRIT
cd PyRIT
uv sync          # creates .venv, installs Python 3.12, installs all deps

# Verify
uv pip show pyrit

# Optional extras
uv sync --extra huggingface   # HuggingFace models
uv sync --extra all           # everything
```

Run scripts and CLI tools through `uv run` to stay in the venv:

```bash
uv run python my_script.py
uv run pyrit_scan --help
uv run pyrit_shell
```

### Jupyter Kernel Setup

```bash
uv run ipython kernel install --user --env VIRTUAL_ENV $(pwd)/.venv --name=pyrit-dev
uv run jupyter lab
```

## Configuration

PyRIT needs two files in `~/.pyrit/`:

```
~/.pyrit/
├── .env          ← API credentials (provider endpoints, keys, model names)
└── .pyrit_conf   ← framework config (database type, initializers, env file paths)
```

### .env File

All providers use the same three variables because `OpenAIChatTarget` works with any OpenAI-compatible API:

```bash
mkdir -p ~/.pyrit
cp .env_example ~/.pyrit/.env   # copy from cloned repo root
# then edit:
```

**Ollama (local):**
```bash
OPENAI_CHAT_ENDPOINT="http://localhost:11434/v1"
OPENAI_CHAT_KEY="ollama"          # any non-empty string; Ollama ignores it
OPENAI_CHAT_MODEL="llama3.1:8b"   # any model you've pulled
```

**OpenAI:**
```bash
OPENAI_CHAT_ENDPOINT="https://api.openai.com/v1"
OPENAI_CHAT_KEY="sk-your-key-here"
OPENAI_CHAT_MODEL="gpt-4o"
```

**Groq (fast cloud inference, good for high-volume attacks):**
```bash
OPENAI_CHAT_ENDPOINT="https://api.groq.com/openai/v1"
OPENAI_CHAT_KEY="gsk_your-key-here"
OPENAI_CHAT_MODEL="llama-3.1-70b-versatile"
```

**Anthropic Claude** — Claude's API is not OpenAI-compatible natively. Options:
1. Use an OpenAI-compatible proxy (e.g., LiteLLM as a local proxy: `litellm --model claude-opus-4-7 --port 8000`)
2. Use PyRIT's `AnthropicChatTarget` class directly in Python (bypasses the env file pattern)

LiteLLM proxy approach for Claude:
```bash
pip install litellm
litellm --model anthropic/claude-opus-4-7 --port 8000

# Then in .env:
OPENAI_CHAT_ENDPOINT="http://localhost:8000/v1"
OPENAI_CHAT_KEY="your-anthropic-api-key"
OPENAI_CHAT_MODEL="claude-opus-4-7"
```

### .pyrit_conf File

```yaml
# ~/.pyrit/.pyrit_conf

memory_db_type: sqlite    # in_memory | sqlite | azure_sql

initializers:
  - name: simple              # baseline defaults for converters, scorers, attack configs
  - name: load_default_datasets  # seed datasets for built-in scenarios
  - name: scorer              # register scorers (refusal, harm-category, Likert, etc.)
  - name: target              # register targets into TargetRegistry
    args:
      tags:
        - default
        - scorer

silent: false
```

**memory_db_type guidance:**
- `in_memory` — no persistence; use for quick notebooks and experiments
- `sqlite` — persists to `~/.pyrit/pyrit.db`; use for all real work
- `azure_sql` — enterprise; requires Azure SQL connection string in env

**Initializer execution order matters** — `target` must run before `scorer` (scorers need targets registered). The order above is correct.

## Python Quick Start

```python
from pyrit.setup import initialize_pyrit_async
from pyrit.setup.initializers import SimpleInitializer
import os

# Quickest: in-memory, no files needed
os.environ["OPENAI_CHAT_ENDPOINT"] = "http://localhost:11434/v1"
os.environ["OPENAI_CHAT_KEY"] = "ollama"
os.environ["OPENAI_CHAT_MODEL"] = "llama3.1:8b"

await initialize_pyrit_async(memory_db_type="InMemory", initializers=[SimpleInitializer()])
```

```python
# From .pyrit_conf (persistent setup)
from pyrit.setup import initialize_from_config_async
await initialize_from_config_async()   # uses ~/.pyrit/.pyrit_conf
```

```python
# With per-run overrides (3-layer precedence)
from pyrit.setup import ConfigurationLoader
config = ConfigurationLoader.load_with_overrides(
    memory_db_type="in_memory",    # override database for this run
    initializers=["simple"],
)
await config.initialize_pyrit_async()
```

## Model Selection — RTX 4090 (24 GB VRAM)

The RTX 4090's 24 GB VRAM determines what Ollama models run locally at full speed. Rule of thumb: Q4-quantized models need roughly **0.5 GB × billions of parameters**.

| Model | VRAM (Q4) | Fits 4090? | Best for |
|-------|-----------|-----------|----------|
| Llama 3.1 8B | ~5 GB | Yes — headroom for batches | Fast attack generation, high-volume probing |
| Mistral 7B | ~4.5 GB | Yes | Lightweight target simulation |
| Llama 3.1 70B | ~38 GB | **No** | Use Groq API instead |
| CodeLlama 34B | ~20 GB | Yes — near limit | Code-aware attack generation |
| Gemma 2 27B | ~15 GB | Yes | Balanced attack/scoring |
| Qwen2.5 32B | ~19 GB | Yes | Strong instruction following |

**Practical recommendation for red-team work:**
- **Attack generation / target simulation:** Llama 3.1 8B or Qwen2.5 32B (local, fast, free)
- **Scoring / judging results:** Claude Opus 4.7 via API (more reliable harm detection, better calibration)

Keep the attacker and judge on different models to avoid the model grading its own outputs.

## Use Cases: Local vs Cloud

| Scenario | Recommended model | Rationale |
|----------|------------------|-----------|
| Generate 1000s of jailbreak variants | Ollama local (8B–13B) | Speed matters; quality threshold is low |
| Red-team a real product for a report | Claude Opus 4.7 API | Nuanced, consistent, citation-quality |
| Simulate a realistic target LLM | Ollama 32B local | Realistic behavior without API costs |
| Score responses for harm/refusal | Claude Sonnet 4.6 API | Reliable harm calibration; use as judge |
| Develop and test new attack strategies | Ollama local | Tight iteration loop, no latency |
| Automated scenario runs (pyrit_scan) | Mix: local attacker + cloud scorer | Best economics at scale |

## CLI Usage

```bash
# Run a built-in scan scenario
uv run pyrit_scan run --config-file ./my_config.yaml --database InMemory

# Override database type at runtime
uv run pyrit_scan run --database sqlite

# Interactive shell
uv run pyrit_shell

# List registered initializers
uv run pyrit list initializers
```

## Configuration Precedence

Later layers override earlier ones:

1. `~/.pyrit/.pyrit_conf` — always loaded if it exists (lowest priority)
2. `--config-file path` — explicit config file
3. CLI flags / Python kwargs — highest priority

## Environment Variable Precedence (within .env loading)

| Priority | Source |
|----------|--------|
| Lowest | System environment variables |
| Medium | `~/.pyrit/.env` |
| Highest | `~/.pyrit/.env.local` (use for temporary overrides without editing .env) |

If you specify `env_files` in `.pyrit_conf`, only those files are loaded — the defaults above are skipped entirely.

## Gotchas & Notes

- **Python 3.12 required.** `uv sync` handles this automatically via `.python-version`.
- **Ollama must be running** before PyRIT starts. `ollama serve` or the Ollama app must be active.
- **Model must be pulled.** `ollama pull llama3.1:8b` before referencing it in `.env`.
- **Initializer order is load-bearing.** `target` before `scorer` — scorers depend on targets being registered.
- **SQLite vs in-memory:** Notebook experiments work fine with `in_memory`; use `sqlite` for any run you want to review later.
- **70B models don't fit the 4090.** Use Groq's free tier (very fast) for 70B inference instead of crashing Ollama on insufficient VRAM.
- **LiteLLM proxy for Claude** adds a local intermediary; if Claude is only your scorer, calling `AnthropicChatTarget` directly in Python is simpler and avoids the extra process.

## Related Pages

- [[attack/ai/prompt_injection]] — technique theory; PyRIT automates the execution
- [[attack/ai/jailbreaking]] — jailbreak families PyRIT can orchestrate
- [[attack/ai/_overview]] — full AI attack surface map
- [[attack/ai/data_poisoning]] — PyRIT complements but does not directly address training-time attacks

## Sources

- raw/pyrit/local_instalation.md
- raw/pyrit/configure_pyrit.md
- raw/pyrit/populating_secrete.md
- raw/pyrit/configuration_file.md
