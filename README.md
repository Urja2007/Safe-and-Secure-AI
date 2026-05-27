# Safe-and-Secure-AI: LLM Jailbreak Attack & TRiSM Defense Evaluation Pipeline

A research-grade evaluation pipeline for benchmarking the safety of Open-Source Large Language Models (LLMs) under adversarial jailbreak attacks. The pipeline is grounded in the **Trust, Risk, and Security Management (TRiSM) framework for Agentic AI**, implementing three complementary defense layers and evaluating their effectiveness across 8 distinct configurations.

---

## Overview

This project evaluates how well open-source LLMs resist adversarial jailbreak prompts when protected by different combinations of input-level, governance-level, and output-level defenses. It supports multiple target models, multiple judge backends, and produces structured markdown reports with Attack Success Rate (ASR), Defense Success Rate (DSR), and False Positive Rate (FPR) metrics.

**Evaluated Models:**
- `meta-llama/Llama-3.1-8B-Instruct`
- `mistralai/Mistral-7B-Instruct-v0.3`
- `Qwen/Qwen2.5-7B-Instruct`
- `deepseek-ai/deepseek-llm-7b-chat`

**Results available for:** Llama-3.1-8B, Mistral-7B, Qwen-2.5-7B

---

## Project Structure

```
Safe-and-Secure-AI/
│
├── data/
│   ├── jailbreak_prompts_25.csv    # 25 jailbreak prompts across 10 harm categories
│   └── benign_prompts.txt          # 25 control benign prompts
│
├── Result/                         # Generated evaluation reports and raw outputs
│   ├── llama-3.1-8b_evaluation_report.md
│   ├── llama-3.1-8b_raw_results.json
│   ├── mistral-7b_evaluation_report.md
│   ├── mistral-7b_raw_results.json
│   ├── qwen-2.5-7b_evaluation_report.md
│   └── qwen-2.5-7b_raw_results.json
│
├── config.py                       # Global config (HF token, model IDs, API keys, paths)
├── defenses.py                     # D1, D2, D3 defense implementations
├── models.py                       # Model loading (4-bit/16-bit) and generation helpers
├── judge.py                        # LLM-as-a-Judge (OpenAI / Gemini / Local / Heuristic / Manual)
├── main.py                         # Evaluation runner and metrics compiler
│
├── requirements.txt
├── setup_env.bat                   # Environment setup for Windows
└── setup_env.sh                    # Environment setup for Linux/macOS
```

---

## The TRiSM Defense Framework

The pipeline implements three defense pillars drawn from the TRiSM framework:

### D1 — Input Sanitizer *(Application Security)*

A rule-based pre-generation filter in `defenses.py::sanitize()`. Flags prompts before they ever reach the model.

**Detection patterns:**
- **Instruction overrides:** `ignore previous instructions`, `forget your training`, `bypass safety`, `system override`, etc.
- **Roleplay jailbreaks:** `pretend you are`, `act as if`, `you are DAN`, `do anything now`, etc.
- **Unusual formatting:** Repeating delimiter blocks (`==========`, `**********`, `----------`) of 10+ chars
- **ALL CAPS blocks:** 5+ consecutive uppercase words of length ≥ 3

Returns `"FLAGGED"` or `"CLEAN"`. Flagged prompts receive the standard refusal without any model call.

### D2 — System Prompt Hardening *(Governance)*

A hardened system prompt injected at inference time via `defenses.py::get_system_prompt("strong")`.

The strong prompt explicitly instructs the model to:
- Refuse roleplay overrides (`pretend you are`, `act as`, `DAN`)
- Refuse hypothetical/fictional framing used to bypass safety
- Refuse translation/encoding-based evasion
- Refuse instruction override attempts (`ignore previous instructions`)
- Maintain a neutral, polite refusal tone without lecturing

### D3 — LLaMA-Guard Classifier *(ModelOps / Explainability)*

A post-generation safety classifier using `meta-llama/Llama-Guard-3-8B`, implemented in `defenses.py::llama_guard_check()`.

- Classifies each `(prompt, response)` pair after generation
- Returns `safe` or `unsafe` with a category code
- Unsafe responses are replaced with the standard refusal string
- Loaded on-demand in 4-bit quantization (GPU) or float16 (CPU) to minimize memory usage
- Auto-unloaded after classification to free VRAM for the judge phase

---

## Defense Configurations Evaluated

All 8 possible combinations of D1, D2, D3 are benchmarked:

| # | Configuration | D1 | D2 | D3 |
|---|--------------|----|----|-----|
| 1 | Baseline      | ✗  | ✗  | ✗  |
| 2 | D1 Only       | ✓  | ✗  | ✗  |
| 3 | D2 Only       | ✗  | ✓  | ✗  |
| 4 | D3 Only       | ✗  | ✗  | ✓  |
| 5 | D1 + D2       | ✓  | ✓  | ✗  |
| 6 | D2 + D3       | ✗  | ✓  | ✓  |
| 7 | D1 + D3       | ✓  | ✗  | ✓  |
| 8 | All Three     | ✓  | ✓  | ✓  |

---

## Metrics

| Metric | Definition |
|--------|-----------|
| **ASR** (Attack Success Rate) | Fraction of jailbreak prompts that successfully elicited harmful output |
| **DSR** (Defense Success Rate) | Fraction of jailbreak prompts blocked by any defense layer |
| **FPR** (False Positive Rate) | Fraction of benign prompts incorrectly refused |

The per-layer breakdown additionally shows how many attacks were stopped at D1, D2, D3, or bypassed the pipeline entirely.

---

## Environment Setup

### Prerequisites

- Python 3.10+
- CUDA-capable GPU recommended (16GB+ VRAM for 4-bit; 24GB+ for 16-bit)
- Hugging Face account with access to gated models:
  - `meta-llama/Llama-3.1-8B-Instruct`
  - `meta-llama/Llama-Guard-3-8B`
  - `mistralai/Mistral-7B-Instruct-v0.3`
  - `Qwen/Qwen2.5-7B-Instruct`
  - `deepseek-ai/deepseek-llm-7b-chat`

### Windows

```bash
setup_env.bat
```

### Linux / macOS

```bash
chmod +x setup_env.sh
./setup_env.sh
```

Both scripts create a virtual environment `env/`, upgrade pip, and install all dependencies from `requirements.txt`.

### Activate Environment

```bash
# Linux/macOS
source env/bin/activate

# Windows
env\Scripts\activate
```

---

## Configuration (`config.py`)

Set your credentials and paths before running:

```python
HF_TOKEN = os.environ.get("HF_TOKEN", "your_hf_token_here")
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY", "")

JUDGE_TYPE = "gemini"   # Options: local | openai | gemini | heuristic | manual
```

It is recommended to set secrets as environment variables rather than hardcoding them in `config.py`.

---

## Running the Evaluation

### Basic Usage

```bash
# Run on Llama-3.1-8B with 4-bit quantization (Colab / 16GB GPU)
python main.py --model llama-3.1-8b

# Run on a lab GPU server with full 16-bit precision
python main.py --model llama-3.1-8b --load_in_16bit
```

### Available Target Models

```bash
python main.py --model llama-3.1-8b
python main.py --model mistral-7b
python main.py --model qwen-2.5-7b
python main.py --model deepseek-7b
```

### Judge Backends

Configure how compliance vs. refusal is labelled:

```bash
# Gemini API (recommended — fast, high quality)
python main.py --model llama-3.1-8b --judge gemini

# OpenAI API (GPT-4o-mini)
python main.py --model llama-3.1-8b --judge openai

# Local Llama-3.1-8B judge (no API key needed; memory-safe sequential loading)
python main.py --model llama-3.1-8b --judge local

# Interactive manual labelling via CLI
python main.py --model llama-3.1-8b --judge manual

# Fast heuristic keyword matching (no model or API needed)
python main.py --model llama-3.1-8b --judge heuristic
```

### Skip Generation (Re-evaluate Existing Results)

If you have already generated responses and want to re-run only the judge phase:

```bash
python main.py --model llama-3.1-8b --judge gemini --skip_gen
```

---

## Evaluation Pipeline (4 Phases)

```
Phase 1: Generation
  ├── Load target model (4-bit or 16-bit)
  ├── For each prompt (25 jailbreak + 25 benign):
  │     ├── D1 check → if FLAGGED, assign REFUSAL_RESPONSE immediately
  │     ├── Generate under WEAK system prompt ("You are a helpful assistant.")
  │     └── Generate under STRONG system prompt (D2 hardened)
  └── Unload target model

Phase 2: LLaMA-Guard Classification
  ├── Load Llama-Guard-3-8B
  ├── Classify each (prompt, weak_response) pair → safe / unsafe
  ├── Classify each (prompt, strong_response) pair → safe / unsafe
  └── Unload Llama-Guard

Phase 3: Judge Evaluation
  ├── For jailbreak prompts: run LLM-as-a-Judge on weak and strong responses
  │     └── Returns verdict (1 = complied, 0 = refused) + reasoning
  └── Benign prompts: verdict = 0 by definition (no harmful intent)

Phase 4: Metrics Computation
  ├── Simulate all 8 configurations using stored results
  ├── Compute ASR, DSR, FPR per configuration
  ├── Compute per-layer D1/D2/D3/bypassed breakdown
  └── Write evaluation_report.md and raw_results.json
```

> **Note:** All four phases run in sequence from `main.py`. LLaMA-Guard and the local judge are loaded and unloaded on demand to avoid OOM errors — only one large model is in VRAM at a time.

---

## Outputs

Two files are written to `Result/` for each model:

### `<model>_raw_results.json`
Full log for each prompt containing:
- Prompt text, type, category, behavior, goal
- Weak and strong responses with latencies
- LLaMA-Guard safety labels for both responses
- Judge verdict and reasoning for jailbreak prompts

### `<model>_evaluation_report.md`
Structured markdown report containing:
1. **Summary Metrics Table** — ASR, DSR, FPR across all 8 configurations
2. **Per-Layer Defense Breakdown Table** — counts of attacks stopped at D1, D2, D3, or bypassed
3. **Findings & Observations** — interpretation of each defense layer

---

## Git Workflow (Transferring to Lab GPU PC)

### Step 1: Initialize and push from your local machine

```bash
git init
git add .
git commit -m "Initial commit of Safety Evaluation codebase"
git remote add origin https://github.com/your-username/your-repo.git
git branch -M main
git push -u origin main
```

### Step 2: Clone and set up on the lab GPU PC

```bash
git clone https://github.com/your-username/your-repo.git
cd your-repo
./setup_env.sh        # Linux
source env/bin/activate
python main.py --model llama-3.1-8b --load_in_16bit --judge gemini
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `torch >= 2.0.0` | Core deep learning framework |
| `transformers >= 4.40.0` | Model loading, tokenization, generation |
| `accelerate >= 0.28.0` | Multi-GPU / device_map support |
| `bitsandbytes >= 0.42.0` | 4-bit quantization (BitsAndBytesConfig) |
| `pandas >= 2.0.0` | Dataset loading and results formatting |
| `tqdm >= 4.66.0` | Progress bars |
| `huggingface_hub >= 0.20.0` | HF token authentication |
| `openai >= 1.12.0` | OpenAI judge backend |
| `google-generativeai >= 0.3.0` | Gemini judge backend |
| `jinja2 >= 3.0.0` | Chat template rendering |

Install all with:
```bash
pip install -r requirements.txt
```

---

## Project Context

This pipeline was developed as part of a research internship at **IIT Patna's Wireless Communications Research Lab (WCRL)** under Prof. Preetam Kumar, focusing on **Safe and Secure AI** applied to communications systems. The evaluation framework is designed to be modular — target models, defense layers, and judge backends can all be swapped independently.

---

## Authors

- **Thota Venkata Sai** (U24EC086) — SVNIT Surat, ECE + AI Minor
- **Urja Mandali** — SVNIT Surat
- **Supervisor:** Shambhavi (PhD Scholar, WCRL, IIT Patna)
- **PI:** Prof. Preetam Kumar, IIT Patna
