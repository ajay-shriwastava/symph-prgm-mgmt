

# Symphony — Agentic AI Orchestration Platform

Vite frontend + FastAPI backend + PostgreSQL database.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML + CSS + JavaScript, served by Vite |
| Backend | FastAPI (Python 3.11+), async SQLAlchemy 2, Alembic |
| Database | PostgreSQL (asyncpg driver) |
| Agent runtime | LangGraph + langchain-anthropic (integrated into FastAPI services) |
| Real-time | FastAPI WebSocket endpoints, ConnectionManager pattern (`app/ws_manager.py`) |

---

## Local Setup

### 1. PostgreSQL — create the database
```bash
brew services restart postgresql
psql -U postgres
CREATE DATABASE symphony;
\q
```

Set the connection string as an environment variable (or rely on the default):

```bash
export DATABASE_URL="postgresql+asyncpg://postgres:postgres@localhost/symphony"
```

---

### 2. Backend (`symph-back-end`)

#### One-time virtualenv setup

```bash
mkvirtualenv symphony
workon symphony
pip install -r requirements.txt
```

#### Install new dependencies (after pulling latest)

```bash
workon symphony
pip install -r requirements.txt
```

#### Run Alembic migration (creates all tables, including workflow_runs)

```bash
workon symphony
alembic upgrade head
```

#### Start the FastAPI dev server

```bash
workon symphony
fastapi dev app/main.py
```

- API: http://127.0.0.1:8000
- Interactive docs: http://127.0.0.1:8000/docs

Shut down with `Ctrl+C`.

---

### 3. Frontend (`symph-front-end`)

```bash
npm install      # first time only
npm run dev
```

- UI: http://localhost:5173/src/html/agents.html

Shut down with `q + Enter` or `Ctrl+C`.

#### Pages

| URL | Purpose |
|---|---|
| /src/html/agents.html | Create, edit, delete agents |
| /src/html/workflows.html | Create, edit, delete workflows |
| /src/html/messages.html | View and filter messages by session or agent |
| /src/html/logs.html | View logs filtered by level, agent, or workflow |
| /src/html/memory.html | Agent Configuration: Memory, Schedules, Skills, Interaction Rules, Guardrails |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost/symphony` | PostgreSQL async connection string |
| `ANTHROPIC_API_KEY` | — | Required for LangGraph agent nodes and Slack bot to call Claude |
| `SLACK_BOT_TOKEN` | — | Slack bot token (`xoxb-...`) for Socket Mode |
| `SLACK_APP_TOKEN` | — | Slack app-level token (`xapp-...`) for Socket Mode |
| `LANGCHAIN_TRACING_V2` | `false` | Set to `true` to enable LangSmith tracing |
| `LANGCHAIN_API_KEY` | — | LangSmith API key |
| `LANGCHAIN_PROJECT` | — | LangSmith project name (e.g. `symphony`) |

---

## Slack Integration

Symphony includes a Socket Mode Slack bot that lets you chat with agents directly from Slack.

### How it works

- The bot listens for **direct messages** and **@mentions**
- On each message it looks up the first agent in the database whose `channels` list includes `"slack"`
- The agent's model and system prompt are used to generate a reply via Claude
- Both the inbound message and the reply are persisted to the `messages` table (grouped by Slack channel as session)
- The bot starts and stops automatically with the FastAPI server (via lifespan)

### Setup

1. Create a Slack app at [api.slack.com/apps](https://api.slack.com/apps) with **Socket Mode** enabled
2. Add bot scopes: `chat:write`, `im:history`, `app_mentions:read`
3. Subscribe to events: `message.im`, `app_mention`
4. Set environment variables:
   ```bash
   export SLACK_BOT_TOKEN="xoxb-..."
   export SLACK_APP_TOKEN="xapp-..."
   ```
5. In the Symphony UI (**Agent Configuration → Channels**), add `slack` to the agent you want to handle Slack messages

If neither token is set, the bot silently disables itself and the rest of Symphony runs normally.

---

## API Overview

All endpoints are under `/api/v1` and require an `Authorization: Bearer <token>` header.
Auth is currently a stub — any non-empty token is accepted.

| Resource | Endpoints |
|---|---|
| Agents | GET/POST `/api/v1/agents`, GET/PUT/DELETE `/api/v1/agents/{id}` |
| Workflows | GET/POST `/api/v1/workflows`, GET/PUT/DELETE `/api/v1/workflows/{id}` |
| Workflow Runs | POST `/api/v1/workflows/{id}/run`, GET `/api/v1/workflows/{id}/runs`, GET `/api/v1/workflows/{id}/runs/{run_id}` |
| Messages | GET/POST `/api/v1/messages`, GET/DELETE `/api/v1/messages/{id}` |
| Logs | GET/POST `/api/v1/logs`, GET `/api/v1/logs/{id}` |
| Agent Memory | GET/POST `/api/v1/agents/{id}/memory`, GET/DELETE `/api/v1/agents/{id}/memory/{key}` |
| Agent Schedules | GET/POST `/api/v1/agents/{id}/schedules`, PUT/DELETE `/api/v1/agents/{id}/schedules/{schedule_id}` |
| Agent Skills | PUT `/api/v1/agents/{id}/skills` |
| Interaction Rules | PUT `/api/v1/agents/{id}/interaction-rules` |
| Guardrails | PUT `/api/v1/agents/{id}/guardrails` |

### WebSocket

| Resource | URL |
|---|---|
| Run event stream | `ws://127.0.0.1:8000/ws/workflows/{workflow_id}/runs/{run_id}?token=<jwt>` |

Events: `node_enter`, `node_complete`, `edge_traverse`, `run_complete`, `run_error`

---

## Observability — LangSmith Tracing

Symphony integrates with [LangSmith](https://smith.langchain.com) for deep LLM tracing of all workflow runs and Slack messages. Tracing is opt-in and requires no code changes — just environment variables.

### Setup

1. Sign up at [smith.langchain.com](https://smith.langchain.com) and create a project named `symphony`
2. Generate an API key under **Settings → API Keys**
3. Add to `symph-back-end/.env`:
   ```
   LANGCHAIN_TRACING_V2=true
   LANGCHAIN_API_KEY=lsv2_pt_...
   LANGCHAIN_PROJECT=symphony
   ```
4. Restart the backend — all subsequent LLM calls are traced automatically

### What gets traced

- **Workflow runs** — full LangGraph execution tree with per-node latency, Claude prompt + response, token counts, and cost
- **Slack messages** — each inbound message and its LLM reply

See the `symph-back-end` Readme for full details on viewing and filtering traces in the LangSmith UI.

---

## References

- [FastAPI documentation](https://fastapi.tiangolo.com/tutorial/)
- [SQLAlchemy async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic documentation](https://alembic.sqlalchemy.org/)
- [Vite documentation](https://vite.dev/guide/)
- [LangSmith documentation](https://docs.smith.langchain.com/)