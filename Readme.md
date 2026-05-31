
# Symphony — Agentic AI Orchestration Platform

Vite frontend + FastAPI backend + PostgreSQL database.

---

## Architecture

```mermaid
graph TB
    Browser(["Browser"])

    subgraph DC ["Docker Compose  (symph-back-end)"]
        direction TB
        nginx["nginx\nport 80"]
        FE["symph-front-end\nVite build · HTML/CSS/JS"]
        BE["symph-back-end\nFastAPI · LangGraph · APScheduler\nport 8000"]
        PG[("PostgreSQL 16\nport 5432")]
        Migrate["alembic upgrade head\n(init container)"]
    end

    Anthropic["☁ Anthropic API\nClaude models"]
    Slack["☁ Slack\nSocket Mode"]
    LangSmith["☁ LangSmith\nTracing (optional)"]

    Browser -->|"HTTP :80  pages + assets"| nginx
    nginx -->|"serves pre-built static files"| FE
    nginx -->|"/api/*  reverse proxy"| BE
    Browser <-->|"WebSocket :8000  live run events"| BE
    Migrate -->|"schema migrations"| PG
    BE -->|"async SQLAlchemy"| PG
    BE -->|"LangGraph agent nodes"| Anthropic
    Slack <-->|"DMs · mentions"| BE
    BE -.->|"LLM traces · token costs"| LangSmith
```

---

## Repositories

| Repo | Purpose |
|---|---|
| **symph-back-end** | FastAPI server — REST API, LangGraph workflow execution, Slack bot, APScheduler cron, Alembic migrations, WebSocket broadcast, Docker Compose entry point |
| **symph-front-end** | Vanilla HTML/CSS/JS frontend — visual workflow builder, agent management, live run panel, Vite dev server (local) / nginx (Docker) |

Both repos must be cloned side-by-side (docker-compose in `symph-back-end` references `../symph-front-end`).

---

## Getting Started

### Prerequisites

| Tool | Version | Install |
|---|---|---|
| Git | any | https://git-scm.com |
| Docker Desktop | 4.x+ | https://www.docker.com/products/docker-desktop |
| An Anthropic API key | — | https://console.anthropic.com |

Docker Desktop must be running before you start.

### 1. Clone the repositories

Symphony consists of two repos that must sit next to each other in the same parent directory:

```bash
mkdir symphony && cd symphony

git clone https://github.com/ajay-shriwastava/symph-back-end.git
git clone https://github.com/ajay-shriwastava/symph-front-end.git
```

Your directory structure should look like this:

```
symphony/
  symph-back-end/     ← FastAPI backend (clone this first)
  symph-front-end/    ← Vite frontend
```

### 2. Configure environment variables

```bash
cd symph-back-end
cp .env.example .env
```

Open `.env` and set your values:

```
ANTHROPIC_API_KEY=sk-ant-...        # required — get from console.anthropic.com
SLACK_BOT_TOKEN=xoxb-...            # optional — Slack integration
SLACK_APP_TOKEN=xapp-...            # optional — Slack integration
SLACK_REPORT_CHANNEL=data-reports   # optional — Slack channel for reports
```

All other values in `.env.example` can be left at their defaults for a local run.

### 3. Start everything

```bash
docker compose up --build
```

This single command:
- Starts PostgreSQL and waits for it to be healthy
- Runs all database migrations automatically (`alembic upgrade head`)
- Builds and starts the FastAPI backend
- Builds the frontend and serves it via nginx

First build takes 2–4 minutes (downloading base images, installing dependencies). Subsequent runs: `docker compose up`.

### 4. Open the app

| URL | What you get |
|---|---|
| http://localhost | Symphony UI |
| http://localhost:8000/docs | Interactive API docs (Swagger UI) |

### 5. Stop

```bash
docker compose down          # stops containers, keeps the database volume
docker compose down -v       # stops containers AND deletes the database
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML + CSS + JavaScript, served by Vite (dev) / nginx (Docker) |
| Backend | FastAPI (Python 3.11+), async SQLAlchemy 2, Alembic |
| Database | PostgreSQL (asyncpg driver) |
| Agent runtime | LangGraph + langchain-anthropic (integrated into FastAPI services) |
| Real-time | FastAPI WebSocket endpoints, ConnectionManager pattern (`app/ws_manager.py`) |
| Messaging | Slack Socket Mode bot (`app/slack_bot.py`) |
| Scheduling | APScheduler cron integration (`app/scheduler.py`) |
| Observability | LangSmith tracing (opt-in via env vars) |
| Containerisation | Docker Compose (single-command setup) |

---

## Quick Start — Docker

```bash
cd symph-back-end
cp .env.example .env          # fill in ANTHROPIC_API_KEY (and optional Slack tokens)
docker compose up --build
```

- **Frontend**: http://localhost
- **API docs**: http://localhost:8000/docs

This single command starts Postgres, runs all Alembic migrations, and brings up the backend and frontend. Subsequent runs need only `docker compose up`.

---

## Local Dev Setup

### 1. PostgreSQL — create the database

```bash
brew services restart postgresql
psql -U postgres -c "CREATE DATABASE symphony;"
```

### 2. Backend (`symph-back-end`)

```bash
mkvirtualenv symphony
workon symphony
pip install -r requirements.txt
alembic upgrade head
fastapi dev app/main.py        # → http://127.0.0.1:8000/docs
```

### 3. Frontend (`symph-front-end`)

```bash
npm install
npm run dev                    # → http://localhost:5173/src/html/agents.html
```

> In local dev without Docker, set `BASE_URL = "http://localhost:8000"` in `symph-front-end/src/js/api.js`.

---

## Features

### Agent Management
Create and configure AI agents with a name, model (Claude Sonnet / Haiku), system prompt, tools, and memory. Each agent can be independently scheduled, given skills, interaction rules, and guardrails, and assigned to messaging channels.

### Visual Workflow Builder
Drag-and-drop SVG canvas on the Workflows page. Supports Start, Agent, Condition, and End nodes connected by bezier edges. Condition nodes support branching (true/false) and feedback loops up to a configurable `max_loops` (default: 20).

### Workflow Execution
Workflows run as LangGraph graphs. Each run is tracked in the `workflow_runs` table with status, input/output, and token usage. Live execution events stream to the browser via WebSocket (`node_enter`, `node_complete`, `edge_traverse`, `run_complete`, `run_error`).

### Workflow Templates
Two pre-built templates available from the Workflows page:

| Template | Schedule | What it does |
|---|---|---|
| Data Ingestion Pipeline | Every minute | Scans for CSVs, checks quality, ingests to DB, profiles data, posts report to Slack |
| SRE Job Summary | Every hour | Queries 24h workflow run stats, writes a health summary, posts to Slack |

### Agent Messaging via Slack
A Socket Mode Slack bot starts automatically with the FastAPI server. It routes DMs and @mentions to the configured agent and persists all messages. Configurable per-agent via Agent Configuration → Channels.

### Agent Configuration
Per-agent settings managed via the UI (memory page):
- **Memory**: persistent key/value store per agent
- **Schedules**: cron-based automatic triggering
- **Skills**: capabilities the agent is allowed to use
- **Interaction Rules**: constraints on how the agent communicates
- **Guardrails**: safety boundaries
- **Channels**: messaging integrations (e.g. `slack`)

### Observability
LangSmith tracing for all LLM calls — full prompt/response, token counts, cost, and per-node latency. Opt-in via environment variables, no code changes needed.

---

## Tests

139 tests (integration + unit) covering all routers, the workflow runner, and agent memory.

```bash
# One-time: create the test database
psql -U postgres -c "CREATE DATABASE symphony_test;"

# Run all tests
workon symphony
pytest

# With coverage
pytest --cov=app --cov-report=term-missing
```

Tests use a dedicated `symphony_test` database (never touches the dev database). All tables are truncated between tests.

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost/symphony` | PostgreSQL async connection string |
| `ANTHROPIC_API_KEY` | — | **Required** for LangGraph agent nodes and Slack bot |
| `SLACK_BOT_TOKEN` | — | Slack bot token (`xoxb-...`) for Socket Mode |
| `SLACK_APP_TOKEN` | — | Slack app-level token (`xapp-...`) for Socket Mode |
| `SLACK_REPORT_CHANNEL` | `data-reports` | Slack channel for pipeline reports |
| `DATASET_DIR` | — | Path to dataset directory (Data Ingestion Pipeline template) |
| `LANGCHAIN_TRACING_V2` | `false` | Set to `true` to enable LangSmith tracing |
| `LANGCHAIN_API_KEY` | — | LangSmith API key |
| `LANGCHAIN_PROJECT` | `symphony` | LangSmith project name |
| `POSTGRES_PASSWORD` | `postgres` | Docker Compose only |

---

## Workflow Templates

### Template 1 — Data Ingestion Pipeline

```
Start → Scan CSV → File Found? (condition)
  [false] → End
  [true]  → Data Quality → Ingest to DB → Data Profile → Report Agent → Publish Report → End
```

**Required env vars:** `DATASET_DIR`, `SLACK_REPORT_CHANNEL`, `ANTHROPIC_API_KEY`

### Template 2 — SRE Job Summary

```
Start → Collect Job Stats → SRE Report Agent → Post to Slack → End
```

**Required env vars:** `SLACK_BOT_TOKEN` (bot must be invited to `#job-summary`)

---

## Slack Integration

Symphony includes a Socket Mode Slack bot that lets you chat with agents directly from Slack.

**Setup:**
1. Create a Slack app at [api.slack.com/apps](https://api.slack.com/apps) with **Socket Mode** enabled
2. Add bot scopes: `chat:write`, `im:history`, `app_mentions:read`
3. Subscribe to events: `message.im`, `app_mention`
4. Set `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` in `.env`
5. In the Symphony UI (**Agent Configuration → Channels**), add `slack` to the target agent

If tokens are not set, the bot silently disables itself and the rest of Symphony runs normally.

---

## Observability — LangSmith Tracing

1. Sign up at [smith.langchain.com](https://smith.langchain.com) and create a project named `symphony`
2. Generate an API key under **Settings → API Keys**
3. Add to `symph-back-end/.env`:
   ```
   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_API_KEY=lsv2_pt_...
   LANGCHAIN_PROJECT=symphony
   ```
4. Restart the backend — all subsequent LLM calls are traced automatically

---

## API Overview

All endpoints under `/api/v1`. Auth is a stub — any non-empty Bearer token is accepted.

| Resource | Endpoints |
|---|---|
| Agents | GET/POST `/agents`, GET/PUT/DELETE `/agents/{id}` |
| Agent Memory | GET/POST `/agents/{id}/memory`, GET/DELETE `/agents/{id}/memory/{key}` |
| Agent Schedules | GET/POST `/agents/{id}/schedules`, PUT/DELETE `/agents/{id}/schedules/{schedule_id}` |
| Agent Skills | PUT `/agents/{id}/skills` |
| Interaction Rules | PUT `/agents/{id}/interaction-rules` |
| Guardrails | PUT `/agents/{id}/guardrails` |
| Workflows | GET/POST `/workflows`, GET/PUT/DELETE `/workflows/{id}` |
| Workflow Runs | POST `/workflows/{id}/run`, GET `/workflows/{id}/runs`, GET `/workflows/{id}/runs/{run_id}` |
| Templates | GET `/templates`, POST `/templates/{id}/instantiate` |
| Messages | GET/POST `/messages`, GET/DELETE `/messages/{id}` |
| Logs | GET/POST `/logs`, GET `/logs/{id}` |
| WebSocket | `ws://localhost:8000/ws/workflows/{id}/runs/{run_id}?token=<token>` |

---

## References

- [FastAPI documentation](https://fastapi.tiangolo.com/tutorial/)
- [SQLAlchemy async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic documentation](https://alembic.sqlalchemy.org/)
- [Vite documentation](https://vite.dev/guide/)
- [LangSmith documentation](https://docs.smith.langchain.com/)
- [LangGraph documentation](https://langchain-ai.github.io/langgraph/)
- [Slack Bolt / Socket Mode](https://slack.dev/bolt-python/concepts)
