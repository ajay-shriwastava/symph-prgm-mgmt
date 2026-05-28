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
  main.py            — FastAPI app + router registration + CORS
  database.py        — async engine, AsyncSessionLocal, Base, get_db()
  dependencies.py    — get_current_user (JWT stub)
  models/            — SQLAlchemy ORM models, __init__.py imports all for Alembic
  schemas/           — Pydantic request/response schemas
  routers/           — APIRouter handlers (agents, workflows, messages, logs, workflow_runs)
  ws_manager.py      — ConnectionManager: connect/disconnect/broadcast per run_id
  workflow_runner.py — WorkflowRunner.compile() + run_workflow() async function
alembic/
  env.py             — async run_migrations pattern, imports app.models
  versions/          — migration files
alembic.ini
requirements.txt
```

## Key Patterns

- Log ORM uses `metadata_` (column alias `metadata`) to avoid SQLAlchemy reserved name conflict. Router manually maps it to `LogOut`.
- AgentMemory upsert: PostgreSQL `INSERT ... ON CONFLICT (agent_id, key) DO UPDATE` via `sqlalchemy.dialects.postgresql.insert`.
- All list endpoints return `{ items, total, skip, limit }` envelope.
- Alembic: `0001_initial_schema.py` creates all 5 original tables. `0003` adds workflow.status + workflow_runs table.
- WebSocket routes need a SEPARATE APIRouter with no prefix (`ws_router`). Export it from the router module and register it in main.py separately. Do NOT put `@router.websocket` on a prefixed APIRouter.
- Background tasks needing DB access: use `asyncio.create_task()` with a new `async with AsyncSessionLocal() as bg_db` — never reuse the request-scoped `db` session after it closes.
- workflow_runs router exports two objects: `router` (prefix `/api/v1/workflows`) and `ws_router` (no prefix, WebSocket at `/ws/workflows/{wf_id}/runs/{run_id}`).

## Frontend Structure (symph-front-end/src)

- `api.js` — shared fetch helper, all entity API functions exported
- `nav.js` — `renderNav(activePage)` + `showToast(msg, type)` exported
- `symphony.css` — shared design tokens, nav, table, form, badge, toast, pagination, builder styles
- One `.html` per entity: agents, workflows, messages, logs, memory
- Pages use `<script type="module">` importing from `./api.js` and `./nav.js`
- DOM text set via `textContent` (not innerHTML) for user/API data

## Completed Features

- **persistence-layer** (2026-05-27): Full CRUD for Agent, Workflow, Message, Log, AgentMemory. PostgreSQL + Alembic + async SQLAlchemy. Frontend tabular views with pagination, inline forms, toast errors.
- **visual-workflow-builder** (2026-05-28): SVG canvas with drag-drop nodes (start/agent/condition/end), cubic bezier edges with arrowheads, config panel, LangGraph runner, WebSocket run streaming, run history panel. Migration 0003 adds `workflow.status` column + `workflow_runs` table.

## API Naming Convention

- Base path: `/api/v1`
- WebSocket base path: `/ws`
- All REST endpoints require `Authorization: Bearer <token>`
- WebSocket auth: `?token=<jwt>` query param
- 404 on missing resource, 422 on Pydantic validation failure, 204 on successful delete

## LangGraph Pattern

- `WorkflowRunner.compile(graph_definition, agents_map, run_id)` returns an uncompiled `StateGraph`
- Call `.compile()` on the returned graph before `.ainvoke(state)`
- State dict keys: `messages` (List[str]), `current_output` (Any), `condition_result` (bool), `run_id` (str)
- Condition nodes use `add_conditional_edges` with a router function reading `state["condition_result"]`
- Agent nodes call `ChatAnthropic(model="claude-haiku-4-5")` with the agent's system_prompt
- `ANTHROPIC_API_KEY` env var required for agent node execution
