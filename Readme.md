

# Symphony — Agentic AI Orchestration Platform

Vite frontend + FastAPI backend + PostgreSQL database.

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | Vanilla HTML + CSS + JavaScript, served by Vite |
| Backend | FastAPI (Python 3.11+), async SQLAlchemy 2, Alembic |
| Database | PostgreSQL (asyncpg driver) |
| Agent runtime | LangGraph (integrated into FastAPI services) |

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

#### Run Alembic migration (creates all tables)

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

- UI: http://localhost:5173/src/agents.html

Shut down with `q + Enter` or `Ctrl+C`.

#### Pages

| URL | Purpose |
|---|---|
| /src/agents.html | Create, edit, delete agents |
| /src/workflows.html | Create, edit, delete workflows |
| /src/messages.html | View and filter messages by session or agent |
| /src/logs.html | View logs filtered by level, agent, or workflow |
| /src/memory.html | Manage key/value memory per agent |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://postgres:postgres@localhost/symphony` | PostgreSQL async connection string |

---

## API Overview

All endpoints are under `/api/v1` and require an `Authorization: Bearer <token>` header.
Auth is currently a stub — any non-empty token is accepted.

| Resource | Endpoints |
|---|---|
| Agents | GET/POST `/api/v1/agents`, GET/PUT/DELETE `/api/v1/agents/{id}` |
| Workflows | GET/POST `/api/v1/workflows`, GET/PUT/DELETE `/api/v1/workflows/{id}` |
| Messages | GET/POST `/api/v1/messages`, GET/DELETE `/api/v1/messages/{id}` |
| Logs | GET/POST `/api/v1/logs`, GET `/api/v1/logs/{id}` |
| Agent Memory | GET/POST `/api/v1/agents/{id}/memory`, GET/DELETE `/api/v1/agents/{id}/memory/{key}` |

---

## References

- [FastAPI documentation](https://fastapi.tiangolo.com/tutorial/)
- [SQLAlchemy async](https://docs.sqlalchemy.org/en/20/orm/extensions/asyncio.html)
- [Alembic documentation](https://alembic.sqlalchemy.org/)
- [Vite documentation](https://vite.dev/guide/)