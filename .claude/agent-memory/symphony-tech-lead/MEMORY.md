# Symphony Tech Lead Memory

## Architecture Decisions

- Sub-agents cannot be invoked as nested Claude Code processes (CLAUDECODE env var blocks nesting). Tech Lead implements all code directly when nested invocation fails.
- Backend virtualenv: `symphony` (mkvirtualenv/workon)
- DB driver: `asyncpg` — DATABASE_URL format: `postgresql+asyncpg://user:pass@localhost/symphony`
- Auth: JWT stub in `app/dependencies.py` — `get_current_user` returns `{"id": "stub-user"}` for any bearer token. Token stored in `localStorage["symphony_token"]`, fallback `"dev-token"`.
- CORS: allow all origins in dev (fastapi CORSMiddleware).

## Backend Structure (symph-back-end)

```
app/
  main.py          — FastAPI app + router registration + CORS
  database.py      — async engine, AsyncSessionLocal, Base, get_db()
  dependencies.py  — get_current_user (JWT stub)
  models/          — SQLAlchemy ORM models, __init__.py imports all for Alembic
  schemas/         — Pydantic request/response schemas
  routers/         — APIRouter handlers (agents, workflows, messages, logs)
alembic/
  env.py           — async run_migrations pattern, imports app.models
  versions/        — migration files
alembic.ini
requirements.txt
```

## Key Patterns

- Log ORM uses `metadata_` (column alias `metadata`) to avoid SQLAlchemy reserved name conflict. Router manually maps it to `LogOut`.
- AgentMemory upsert: PostgreSQL `INSERT ... ON CONFLICT (agent_id, key) DO UPDATE` via `sqlalchemy.dialects.postgresql.insert`.
- All list endpoints return `{ items, total, skip, limit }` envelope.
- Alembic: `0001_initial_schema.py` creates all 5 tables with correct indexes and FK constraints.

## Frontend Structure (symph-front-end/src)

- `api.js` — shared fetch helper, all entity API functions exported
- `nav.js` — `renderNav(activePage)` + `showToast(msg, type)` exported
- `symphony.css` — shared design tokens, nav, table, form, badge, toast, pagination styles
- One `.html` per entity: agents, workflows, messages, logs, memory
- Pages use `<script type="module">` importing from `./api.js` and `./nav.js`
- DOM text set via `textContent` (not innerHTML) for user/API data

## Completed Features

- **persistence-layer** (2026-05-27): Full CRUD for Agent, Workflow, Message, Log, AgentMemory. PostgreSQL + Alembic + async SQLAlchemy. Frontend tabular views with pagination, inline forms, toast errors.

## API Naming Convention

- Base path: `/api/v1`
- All endpoints require `Authorization: Bearer <token>`
- 404 on missing resource, 422 on Pydantic validation failure, 204 on successful delete
