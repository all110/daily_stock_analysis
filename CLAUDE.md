# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Governance

`AGENTS.md` is the canonical AI collaboration rulebook for this repository — it defines the hard rules, verification matrix, PR workflow, and delivery standards. When any instruction here conflicts with `AGENTS.md`, `AGENTS.md` wins.

## Project overview

AI-driven stock analysis system covering A-shares (CN), HK, US, JP (`.T`), and KR (`.KS`/`.KQ`) markets. The pipeline fetches market data, runs technical/LLM analysis, renders reports via Jinja2, and pushes notifications to 12+ channels.

## Common commands

### Running the app

```bash
python main.py                          # full analysis run
python main.py --debug                  # debug logging
python main.py --dry-run                # skip notifications
python main.py --stocks 600519,hk00700,AAPL   # specific stocks
python main.py --market-review          # market-wide review
python main.py --schedule               # recurring scheduled mode
python main.py --serve                  # FastAPI server + scheduled analysis
python main.py --serve-only             # FastAPI server only
python main.py --webui                  # serve + web UI
python main.py --webui-only             # serve + web UI, no analysis
uvicorn server:app --reload --host 0.0.0.0 --port 8000
```

### Backend validation

```bash
pip install -r requirements.txt
./scripts/ci_gate.sh                    # full CI gate: syntax + flake8 + deterministic + offline tests
./scripts/ci_gate.sh syntax             # syntax check only
python -m pytest -m "not network"       # offline tests (skip network-dependent)
python -m py_compile <changed_file.py>  # single-file syntax check
flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics  # critical lint only
```

### Web frontend (`apps/dsa-web/`)

```bash
cd apps/dsa-web
npm ci && npm run lint && npm run build   # full validation
npm test                                  # Vitest unit tests
npm run test:smoke                        # Playwright E2E
```

### Desktop app (`apps/dsa-desktop/`)

```bash
cd apps/dsa-desktop
npm install && npm run build
npm test                                  # Node --test
```

### Docker

```bash
docker build -f docker/Dockerfile -t dsa .
docker compose -f docker/docker-compose.yml up
```

### PR/CI evidence

```bash
gh pr view <pr_number>
gh pr checks <pr_number>
gh run view <run_id> --log-failed
```

## Architecture

### Analysis pipeline (the core loop)

```
main.py  →  setup_env()  →  parse stocks  →  src/core/pipeline.py
                                                    │
   data_provider/*_fetcher.py  ←── fetch quotes/technicals/news/fundamentals
   src/llm/                    ←── LLM analysis (via litellm + local CLI backends)
   src/services/               ←── scoring, alerts, decision signals
   templates/*.j2              ←── Jinja2 report rendering
   src/notification_sender/*   ←── push to channels (WeChat, Feishu, Telegram, ...)
```

### Directory responsibilities

| Directory | Role |
|---|---|
| `src/core/` | Pipeline orchestration, backtest engine, trading calendar, config management |
| `src/services/` | Business logic layer (37 services): analysis, scoring, alerts, portfolios, scheduling, import |
| `src/repositories/` | SQLAlchemy data access layer, one repo per domain aggregate |
| `src/schemas/` | Pydantic models for reports, decision actions, analysis context, market light |
| `src/llm/` | LLM abstraction: litellm backend (Gemini/Anthropic/OpenAI/DeepSeek), local CLI backend, factory, provider cache, usage tracking |
| `src/agent/` | Multi-agent chat system: orchestrator, executor, factory, memory, conversation, research, runner; subdirs for agents/skills/strategies/tools |
| `src/notification_sender/` | 13 notification channel senders, each in its own file |
| `data_provider/` | Multi-source data fetchers with graceful fallback (efinance → akshare → tushare → pytdx → baostock → yfinance → longbridge) |
| `api/` | FastAPI v1 REST API: endpoints for analysis, agent, stocks, auth, portfolios, backtests, alerts, system config, health |
| `bot/` | Chatbot dispatcher: platform adapters (DingTalk, Discord, Feishu) + commands (analyze, ask, chat, market, research, history, strategies) |
| `apps/dsa-web/` | React 19 + Vite 7 + Tailwind 4 + Zustand + Recharts frontend |
| `apps/dsa-desktop/` | Electron wrapper around the web app, built with electron-builder |
| `strategies/` | 15 YAML strategy definitions (chan theory, wave theory, MA crossover, event-driven, etc.) |
| `templates/` | Jinja2 templates for Markdown, WeChat, and brief report formats |
| `tests/` | pytest suite with `unit`, `integration`, and `network` markers |
| `docker/` | Multi-stage Dockerfile (Node frontend build → Python 3.11-slim) + compose |

### Key architectural patterns

- **Config**: Single large `src/config.py` (146KB) loads `.env` via python-dotenv and writes to `os.environ`. All env vars prefixed with `STOCK_`, `LLM_`, `NOTIFY_`, etc. New config should default to "works without it, enhanced with it."
- **Provider fallback chain**: `data_provider/base.py` defines the fallback order. Each fetcher is self-contained in its own file. Single provider failure must not crash the pipeline.
- **Notification resilience**: Each channel sender is independent; a single channel failure must not block other channels or the analysis pipeline.
- **LLM abstraction**: `src/llm/backend_factory.py` routes model names to backends. `litellm_backend.py` handles most cloud providers; `local_cli_backend.py` handles local/CLI models (Ollama, codex_cli).
- **Agent system**: `src/agent/orchestrator.py` coordinates multi-agent conversations. Agents are defined in `src/agent/agents/`, skills in `src/agent/skills/`, tools in `src/agent/tools/`, and strategies reference `strategies/*.yaml`.
- **API design**: FastAPI v1 under `api/v1/endpoints/` with Pydantic schemas in `api/v1/schemas/`. Auth middleware in `api/middlewares/auth.py`. Dependency injection via `api/deps.py`.
- **Testing**: pytest with markers `unit` (fast, offline), `integration` (service-level, offline), `network` (external services). CI runs `--markers "not network"` by default; `network` tests run separately as observation, not gating.

### Entry points beyond main.py

- `server.py` — ASGI app factory used by `uvicorn server:app`
- `webui.py` — Web UI launcher
- `main.py` — Unified CLI with mutually exclusive modes: one-shot analysis, scheduled, serve, webui
- `.github/workflows/00-daily-analysis.yml` — scheduled GitHub Actions run (weekdays 18:00 Beijing time)

### CI pipeline (`.github/workflows/ci.yml`)

1. `ai-governance` — validates AGENTS.md/CLAUDE.md/.github instructions/.claude skills consistency (blocking)
2. `backend-gate` — runs `./scripts/ci_gate.sh` (blocking)
3. `docker-build` — Docker build + key module import smoke test (blocking)
4. `web-gate` — `npm run lint` + `npm run build` on frontend changes (blocking when triggered)
5. `network-smoke` — `pytest -m network` (observation only, not blocking)
6. `pr-review` — automated PR review + labeling (assistive, not blocking)

### Auto-tag behavior

Opt-in: only commits whose title contains `#patch`, `#minor`, or `#major` trigger version bumps.
