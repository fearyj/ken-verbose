---
title: "VulnDetect AI — Using LLMs to Cut False Positives in Static Analysis"
slug: "vulndetect-ai"
date: "2026-03-12"
description: "My final year project: combining CodeQL with a LangGraph multi-agent pipeline and local LLMs to reduce false positives in C/C++ vulnerability detection."
tags: ["security", "LLM", "static-analysis", "final-year-project"]
---

## Context

This is my **final year project**. The goal was to explore whether LLMs could meaningfully reduce false positives in static analysis — a real pain point in application security that costs teams hours of manual triage.

## The Problem

Static analysis tools like CodeQL are great at finding potential vulnerabilities in code. The problem? They cry wolf — a lot. When you run CodeQL's security-extended suite against a C/C++ codebase, a significant portion of the flagged findings are false positives. This leads to alert fatigue, wasted triage time, and real vulnerabilities getting buried in noise.

I wanted to see if LLMs could help — not by replacing static analysis, but by acting as an intelligent second pass that filters out the noise.

## What I Built

**VulnDetect AI** is a full-stack system that pipes CodeQL results through a multi-agent LangGraph pipeline. Each vulnerability flagged by CodeQL gets analyzed by a chain of specialized agents that gather context, retrieve documentation, and ultimately ask a local LLM: *"Is this actually a vulnerability, or is CodeQL wrong?"*

## Tech Stack

**Static Analysis:**
- `CodeQL v2.20.1` — security-extended query suite (454 queries, 20+ CWEs)

**LLM / Agent Layer:**
- `LangGraph` — multi-agent orchestration with shared state machine
- `Ollama` — local LLM inference (Qwen3:14b, LLaMA 3.1:70b, DeepSeek-R1:70b, gpt-oss:20b, Magistral:24b)

**Code Intelligence:**
- `Tree-sitter` — AST parsing for code context extraction
- `DuckDuckGo Search` — CWE context and function documentation retrieval

**Backend:**
- `FastAPI` — REST API with WebSocket support for real-time progress
- `PostgreSQL` — scan metadata and vulnerability results
- `MinIO` — object storage for uploaded source files
- `Redis` — caching documentation lookups
- `SQLAlchemy` + `Alembic` — ORM and migrations
- `Docker Compose` — containerized deployment

**Frontend:**
- `React 19` + `Vite` — UI framework and build tool
- `TailwindCSS` — styling
- `Recharts` — benchmark result visualization
- `React Router v7` — client-side routing

## Why These Choices

**CodeQL v2.20.1** — Industry-standard static analysis from GitHub. I used the `cpp-security-extended.qls` query suite, which covers 20+ CWE categories. I pinned the version to v2.20.1 to match the CASTLE benchmark exactly, ensuring fair comparison.

**LangGraph** — I needed a way to orchestrate multiple analysis steps with shared state. LangGraph's state machine model was a natural fit — each node in the pipeline reads from and writes to a shared `AnalysisState`, and the graph handles the sequencing. I considered a simple function chain, but LangGraph made it easier to add conditional edges, toggle nodes on/off, and visualize the workflow.

**Ollama** — The key constraint was that security-sensitive source code should never leave the network. Ollama lets you run LLMs locally with an OpenAI-compatible API. I tested multiple models: Qwen3:14b, LLaMA 3.1:70b, DeepSeek-R1:70b, gpt-oss:20b, and Magistral:24b — each with different precision/recall tradeoffs.

**Tree-sitter** — I needed to extract precise code context around flagged lines. Tree-sitter gives you a real AST, so I could reliably extract the containing function, variable assignments, and call sites — not just dumb line-range slicing.

**FastAPI + WebSockets** — The LLM analysis takes time (up to 5 minutes per scan). WebSockets let me stream real-time progress updates to the frontend — which node is running, what the LLM is "thinking", and how far along the scan is.

**React + Vite + Tailwind** — Standard modern frontend stack. Vite for fast dev builds, Tailwind for rapid UI iteration. The frontend includes a benchmark dashboard with Recharts for visualizing results across models.

**PostgreSQL + MinIO + Redis** — PostgreSQL for scan metadata and vulnerability results, MinIO for storing uploaded source files, Redis for caching documentation lookups so repeated function queries don't hit web search again.

**DuckDuckGo Search** — For retrieving CWE context and function documentation from cppreference.com and Linux man pages. This is toggleable — the ablation study showed it reduces false positives by 2-6 per model.

## The Pipeline

The core of the system is a 6-node LangGraph state machine. When CodeQL flags a line of code, the pipeline does this:

1. **Extract Code Context** — Tree-sitter parses the AST and extracts a 20-line window around the vulnerable line, plus the containing function, variable assignments, and function calls.

2. **Fetch Documentation** — For each function involved, we search cppreference.com and man pages via DuckDuckGo. If web search is disabled or fails, the LLM generates documentation from its own knowledge.

3. **Analyze Return Values** — Checks how return values are being validated. Many false positives come from CodeQL not recognizing that the developer *did* handle the error case.

4. **Web Search CWE Context** — Maps the CodeQL rule to its CWE number (e.g., `cpp/sql-injection` → CWE-89) and fetches real-world context about the vulnerability class.

5. **LLM Verification** — Sends the annotated source file, all gathered context, and a structured prompt to the LLM. The prompt includes the flagged line, function documentation, return value analysis, and CWE context.

6. **Make Decision** — Parses the LLM's structured JSON response to extract: true/false positive classification, confidence score, reasoning, and remediation recommendations.

## Results

I evaluated the system against the **CASTLE benchmark** — 250 C files across 25 CWEs. The numbers tell an interesting story:

### CASTLE Benchmark Results — All Configurations

| Method | TP | FP | TN | FN | Prec. | Rec. | F1 |
|---|---|---|---|---|---|---|---|
| CodeQL only (baseline) | 39 | 55 | 81 | 111 | 41.5% | 26.0% | 32.0% |
| + Qwen3:14b | 35 | 32 | 88 | 115 | 52.2% | 23.3% | 32.3% |
| + Llama3.1:70b | 31 | 27 | 92 | 119 | 53.4% | 20.7% | 29.8% |
| + DeepSeek-R1:70b | 35 | 27 | 89 | 115 | 56.5% | 23.3% | 33.0% |
| + gpt-oss:20b | 35 | 25 | 90 | 115 | 58.3% | 23.3% | **33.3%** |
| + Magistral:24b | 25 | 3 | 98 | 125 | **89.3%** | 16.7% | 28.1% |

All LLM configurations used confidence threshold = 0.70, no web search.

Every model improved precision over CodeQL's baseline 41.5%. **Magistral:24b** was the most aggressive filter — it pushed precision to **89.3%** by removing 52 of 55 false positives, but at the cost of also dropping 14 true positives. **gpt-oss:20b** achieved the best F1 balance at **33.3%**, removing 30 FPs while only losing 4 TPs. Interestingly, Llama3.1:70b (the largest model) performed worse than smaller models like Qwen3:14b — model size alone doesn't determine filter quality.

### Web Search Ablation

I ran each model with and without the web search node (Node 4: CWE context retrieval) to measure its impact:

| Model | TP (no web) | FP (no web) | TP (web) | FP (web) | ΔFP | ΔTP |
|---|---|---|---|---|---|---|
| Qwen3:14b | 35 | 32 | 34 | 28 | −4 | −1 |
| Llama3.1:70b | 31 | 27 | 31 | 26 | −1 | 0 |
| DeepSeek-R1:70b | 35 | 27 | 36 | 25 | −2 | +1 |
| gpt-oss:20b | 35 | 25 | 37 | 19 | **−6** | **+2** |
| Magistral:24b | 25 | 3 | 26 | 2 | −1 | +1 |

Web search consistently reduced false positives across all models (−1 to −6 FP), and for DeepSeek, gpt-oss, and Magistral it actually *increased* true positives too. **gpt-oss:20b benefited the most** — gaining 2 TPs while dropping 6 FPs with web search enabled. This shows that external CWE context helps the model make better-calibrated decisions, especially for models that are already good at the task.

## Problems I Faced

**LLM response parsing was unreliable.** Even with structured JSON prompts, models would sometimes return malformed JSON, mix reasoning into the JSON fields, or wrap the response in markdown code blocks. I ended up writing a multi-layer parser: try JSON extraction first, fall back to regex-based field extraction, then fall back to keyword detection ("false positive", "not a vulnerability"). When all parsing fails, the system defaults to treating the finding as a true positive — the safe choice.

**CodeQL database creation is slow and brittle.** Creating a CodeQL database for even a small C project can take 30+ seconds, and it's sensitive to compiler flags. I had to use permissive compilation flags to allow vulnerable code (which by definition has issues) to compile without aborting the analysis. Getting this right for the 250 CASTLE benchmark files took significant debugging.

**Model inconsistency across runs.** The same model would sometimes give different verdicts on the same vulnerability across runs, especially at lower confidence levels. This made benchmarking tricky — I had to run multiple passes and look at aggregate trends rather than individual results.

**Web search rate limiting.** DuckDuckGo would occasionally throttle requests during large benchmark runs (250 files × multiple search queries each). I added Redis caching and retry logic with backoff, but it still slowed down full benchmark runs significantly.

**Balancing precision vs. recall.** The models that were best at catching false positives (high precision) tended to be too aggressive and also dismissed real vulnerabilities (low recall). Tuning the prompt to find the right balance was an iterative process — I went through dozens of prompt variations before landing on one that worked reasonably well across models.

**VRAM management.** Running 70B parameter models on a single GPU meant constant memory pressure. Ollama handles model loading/unloading, but switching between models during benchmarking would sometimes cause OOM errors. I had to stagger runs and explicitly unload models between benchmark passes.

## Design Decisions

**Why local LLMs?** Security-sensitive code shouldn't leave your network. Running everything through Ollama means the source code never hits an external API. This matters for enterprise adoption.

**Why a multi-agent pipeline instead of one big prompt?** Each node gathers a specific type of context. This makes the system modular — you can toggle web search on/off, swap LLMs, adjust context windows — and makes it easier to debug which stage is contributing to or hurting accuracy.

**Graceful degradation.** If Ollama goes down, the system doesn't crash — it falls back to treating every CodeQL finding as a true positive (the safe default) and flags that it's running in degraded mode.

**Conservative by default.** When the LLM response can't be parsed or confidence is low, the system marks the finding as a true positive. It's better to surface a false positive than to suppress a real vulnerability.

## What's Next

- Testing against larger, real-world codebases beyond micro-benchmarks
- Experimenting with fine-tuning smaller models on vulnerability classification data
- Adding support for more languages beyond C/C++
- Integrating directly into CI/CD pipelines as a PR check

The code is available on [GitHub](https://github.com/fearyj/vulndetect-ai).
