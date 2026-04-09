---
title: Stock Investment Agent Environment
emoji: 📈
colorFrom: blue
colorTo: green
sdk: docker
app_port: 8000
tags:
  - openenv
  - reinforcement-learning
  - docker
---

# Stock Investment Agent Environment

**OpenEnv**-compatible environment for agents that perform **buy-side style portfolio research**: benchmark-relative stances, per-name risk budgets, written theses on critical situations, and **hedge/no-hedge** calls under macro and liquidity stress. Submitted for the **Scaler School of Technology × Meta / PyTorch OpenEnv Hackathon**.

## Why this environment

Human portfolio workflows combine **classification** (how much conviction?), **risk framing** (defensive vs aggressive sleeve), **narrative justification** (thesis quality), and **tail-risk mechanics** (when an overlay is warranted). This repo encodes those dimensions across **four graded tasks** with **deterministic programmatic graders**, dense **step-level reward signal**, and a **reproducible LLM baseline** (`inference.py`).

## OpenEnv compliance

| Piece | Details |
|-------|---------|
| Manifest | [`openenv.yaml`](openenv.yaml) — `name: stock-investment-agent`, FastAPI `app: server.app:app`, port **8000** |
| Typed models | [`models.py`](models.py) — Pydantic **Action**, **Observation**, **State** (OpenEnv base types when `openenv-core` is installed) |
| API | `POST /reset` → initial observation · `POST /step` → `{ observation, reward, done, info }` · `GET /state` → episode metadata |
| Validation | From repo root: `pip install openenv-core` then `openenv validate` |

## Tasks (easy → expert)

Each episode is **one instrument per step** until all rows are decided or `max_steps` is reached.

| Task | Difficulty | Rows | Agent must output |
|------|------------|------|-------------------|
| `basic_screen` | Easy | 5 | `instrument_id`, `decision` ∈ `overweight` · `neutral` · `underweight` |
| `sector_rotation` | Medium | 10 | Above + `risk_tier` ∈ `defensive` · `balanced` · `aggressive` + **thesis** on **`sr3`** |
| `risk_budget` | Hard | 15 | Above + **theses** on **`rb4`**, **`rb9`**, **`rb12`** |
| `macro_stress` | Expert | 12 | Above + `hedge_recommended` **boolean** + **thesis** on **`ms7`** |

Graders implement **partial credit**, an **ordering bonus** for handling **priority / tail-risk** names earlier, **keyword-based thesis scoring** with **anti-stuffing** checks, and **−0.05** if the agent tries to decide the same `instrument_id` twice. Episode-level **`cumulative_reward`** in `GET /state` is clamped to **(0.001, 0.999)** when `done` for stable reporting in **(0, 1)**.

## Quick start (local)

**Python 3.10+**

```bash
pip install -e .
uvicorn server.app:app --host 0.0.0.0 --port 8000
```

With **[uv](https://github.com/astral-sh/uv)** (lockfile in repo):

```bash
uv sync
uv run uvicorn server.app:app --host 0.0.0.0 --port 8000
```

## Docker

```bash
docker build -t stock-investment-agent .
docker run -p 8000:8000 stock-investment-agent
```

Health: `GET /health` returns `{"status":"healthy"}` (used by the image `HEALTHCHECK`).

## Hugging Face Spaces

This repo is configured for a **Docker Space** (`sdk: docker` in this file’s frontmatter). After deploy, point clients at the Space **API base URL** (often a `*.hf.space` host).

- Set **`ENV_URL`** for `inference.py` to that base (no trailing slash), e.g. `https://YOUR-SPACE.hf.space`.
- Automated checks often **`POST`** to **`{BASE}/reset`** with JSON body and expect **HTTP 200**.

## Baseline inference (`inference.py`)

Mandatory variables for the **OpenAI-compatible** client:

| Variable | Role |
|----------|------|
| `API_BASE_URL` | Chat Completions base URL (e.g. OpenAI, Groq, or HF router). |
| `MODEL_NAME` | Model id for that endpoint. |
| `HF_TOKEN` | API key (fallbacks: `OPENAI_API_KEY`, `GROQ_API_KEY` for local convenience). |
| `ENV_URL` | Running environment base URL (default `http://localhost:8000`). |

```bash
export API_BASE_URL="https://api.openai.com/v1"
export MODEL_NAME="gpt-4o-mini"
export HF_TOKEN="your-key"
export ENV_URL="http://localhost:8000"
python inference.py
```

**Stdout format** (do not change field names or line shape if your evaluator parses logs):

- `[START] task=… env=… model=…`
- One `[STEP] step=… action=… reward=… done=… error=…` per `step`
- `[END] success=… steps=… score=… rewards=…` once per task (always emitted via `finally`)

Target **runtime** under typical hackathon limits (order **tens** of LLM calls; keep provider latency reasonable).

### Reference scores

Run `inference.py` once, then copy **`score=`** from each **`[END]`** line into the table so the README matches reproducible logs.

| Task | Model | API host (redacted) | Date (UTC) | Score |
|------|-------|---------------------|------------|-------|
| `basic_screen` | llama-3.3-70b-versatile | Groq OpenAI-compat | 2026-04-10 | 0.40 |
| `sector_rotation` | llama-3.3-70b-versatile | Groq OpenAI-compat | 2026-04-10 | 0.18 |
| `risk_budget` | llama-3.3-70b-versatile | Groq OpenAI-compat | 2026-04-10 | 0.31 |
| `macro_stress` | llama-3.3-70b-versatile | Groq OpenAI-compat | 2026-04-10 | 0.34 |

\*Last filled from a local run (`ENV_URL=http://127.0.0.1:8000`) where the Groq endpoint returned **401** (no API key in that environment), so each step used `inference.py`’s built-in **neutral / balanced / no-hedge** fallback. Set `GROQ_API_KEY` or `HF_TOKEN` and re-run to record scores from the actual model; against a deployed Space, set `ENV_URL` to `https://<your-space>.hf.space` (no trailing slash).

## HTTP API

| Method | Path | Description |
|--------|------|-------------|
| POST | `/reset` | Body: `{"task_name":"<name>","seed":null}` — starts episode, returns `observation`. |
| POST | `/step` | Body: JSON matching **Investment** action fields (see tasks above). |
| GET | `/state` | Progress and `cumulative_reward`. |
| GET | `/health` | Liveness. |
| GET | `/info` | Task list and metadata. |

## Observation and action (concise)

- **Observation:** `instruments[]` (id, symbol, company, sector, headline, narrative, as_of), `pending_ids`, `decided` (history), `task_*` fields, `step` / `max_steps`, `done`, `episode_id`.
- **Action (JSON):** at minimum `instrument_id` + `decision`; add `risk_tier`, `thesis`, and `hedge_recommended` when the task requires them (see task table).

## Python client

Async helper in [`client.py`](client.py):

```python
import asyncio
from client import StockInvestmentEnv, InvestmentAction

async def main():
    async with StockInvestmentEnv(base_url="http://localhost:8000") as env:
        r = await env.reset("basic_screen")
        r = await env.step(InvestmentAction(instrument_id="s1", decision="underweight"))
        print(r.reward, r.done)

asyncio.run(main())
```

## Repository layout

```
├── openenv.yaml          # OpenEnv manifest
├── Dockerfile
├── pyproject.toml        # Dependencies (+ uv.lock)
├── models.py             # Pydantic Action / Observation / State
├── client.py             # Async HTTP client
├── inference.py          # LLM baseline (hackathon submission script)
└── server/
    ├── app.py            # FastAPI app
    ├── environment.py    # reset / step / state
    ├── graders.py        # Task graders + ordering / thesis logic
    └── tasks.py          # Scenarios and ground truth
```

## Design choices (limitations)

Scenarios are **fixed and hand-written** so **graders stay deterministic** and submissions stay **auditable**—a good fit for **benchmarking LLM agents**, less so for million-step RL with procedural universes. Extensions (e.g. sampled fundamentals with calibrated grader bands) are possible future work if reproducibility can be preserved.

## License

MIT
