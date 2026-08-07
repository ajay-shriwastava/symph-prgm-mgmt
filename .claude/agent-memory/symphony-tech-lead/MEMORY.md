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
  routers/           — APIRouter handlers (agents, agent_config, workflows, messages, logs, workflow_runs)
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
- Alembic: `0001_initial_schema.py` creates all 5 original tables. `0003` adds workflow.status + workflow_runs table. `0004` adds skills/interaction_rules/guardrails JSONB columns to agents + creates agent_schedules table. Latest migration is `0012_message_log_level.py` — always check `alembic/versions/` before naming a new migration, as the spec-provided name may be stale.
- WebSocket routes need a SEPARATE APIRouter with no prefix (`ws_router`). Export it from the router module and register it in main.py separately. Do NOT put `@router.websocket` on a prefixed APIRouter.
- Background tasks needing DB access: use `asyncio.create_task()` with a new `async with AsyncSessionLocal() as bg_db` — never reuse the request-scoped `db` session after it closes.
- workflow_runs router exports two objects: `router` (prefix `/api/v1/workflows`) and `ws_router` (no prefix, WebSocket at `/ws/workflows/{wf_id}/runs/{run_id}`).

## Frontend Structure (symph-front-end/src)

- React 19 + TypeScript SPA, React Router v7, Vite
- `js/api.ts` — all TypeScript interfaces + `apiFetch<T>()` + `WS_BASE`; one function per API endpoint
- `config.ts` — `PAGE_SIZE`, `MODEL_OPTIONS`, `CHANNELS`, `CHANNEL_LABELS`, `AUTH_TOKEN_KEY`, `DEV_TOKEN`, `TOAST_DURATION_MS`
- `App.tsx` — all routes lazy-loaded via `React.lazy()`, wrapped in `ErrorBoundary` + `Suspense`
- `pages/` — one `.tsx` file (or subfolder) per route: `Agents.tsx`, `Workflows.tsx`, `Messages.tsx`, `Logs.tsx`, `AgentConfig.tsx`
- `pages/workflows/` — `WorkflowBuilder.tsx`, `NodeConfigPanel.tsx`, `TemplatesSection.tsx`, `graph-helpers.ts`
- `pages/agent-config/` — `MemoryTab.tsx`, `SchedulesTab.tsx`, `SkillsTab.tsx`, `InteractionRulesTab.tsx`, `GuardrailsTab.tsx`, `types.ts`
- `components/` — `Nav.tsx`, `Pagination.tsx`, `LoadingRows.tsx`, `ErrorBoundary.tsx`
- `hooks/useApiList.ts` — generic paginated list hook: `useApiList<T>(fetcher, limit)` → `{ items, total, skip, loading, setSkip, reload }`
- `context/ToastContext.tsx` — `ToastProvider` + `useToast()` hook
- `utils/truncate.ts` — `truncate(str, n)`
- JSX escaping prevents XSS by default; no `dangerouslySetInnerHTML`

## Completed Features

- **persistence-layer** (2026-05-27): Full CRUD for Agent, Workflow, Message, Log, AgentMemory. PostgreSQL + Alembic + async SQLAlchemy. Frontend tabular views with pagination, inline forms, toast errors.
- **visual-workflow-builder** (2026-05-28): SVG canvas with drag-drop nodes (start/agent/condition/end), cubic bezier edges with arrowheads, config panel, LangGraph runner, WebSocket run streaming, run history panel. Migration 0003 adds `workflow.status` column + `workflow_runs` table.
- **agent-configuration** (2026-05-29): Expanded memory.html into 5-tab Agent Config page (Memory, Schedules, Skills, Interaction Rules, Guardrails). Migration 0004 adds 3 JSONB columns to agents + agent_schedules table. New router: `app/routers/agent_config.py`. Nav label "Memory" renamed to "Config". AgentOut schema extended with skills/interaction_rules/guardrails fields.
- **agent-to-agent-handoffs** (2026-05-31): No DB migration. `messages.role` Literal extended to include `"agent"`. `GET /api/v1/messages` gains `role` filter query param. `workflow_runner.py` `_make_agent_node()` persists agent output to `messages` table (role=agent, session_id=run_id, agent_id=source agent) after each agent node completes — fire-and-forget with exception swallowing. `messages.html` gains "Agent Handoffs" tab with styled cards (agent name resolved from agent map, session truncated, timestamp). `api.js` `getMessages` updated to pass `role` param.
- **full-message-capture** (2026-08-04): Migration 0011 adds `destination_type VARCHAR(50)` + `destination_ref TEXT` (both nullable) to messages. `MessageCreate` gains optional `destination_type: Optional[Literal[...]]` + `destination_ref`. `MessageOut` gains same fields. `_make_agent_node()` gains `next_node_type` + `next_agent_id` params; old single-record agent handoff block replaced with full 4-record block (system/user/assistant-or-tool/agent) in one AsyncSessionLocal. `WorkflowRunner.compile()` computes next-node type from `out_edges` before calling `_make_agent_node`. Frontend: `Message` interface gains `destination_type` + `destination_ref`. Messages page AllMessages tab gains Dest Type (badge) + Dest Ref (truncated) columns; colSpan updated 6→8. DEST_BADGE map added.
- **message-log-level** (2026-08-04): Migration 0012 adds `message_log_level VARCHAR(20)` nullable to agents. `AgentCreate`, `AgentUpdate`, `AgentOut` schemas gain `message_log_level: Optional[Literal['MINIMAL','STANDARD','VERBOSE']] = None`. `workflow_runner.py` adds `_LEVEL_RANK` dict + `_effective_log_level(agent_obj)` (reads `MESSAGE_LOG_LEVEL` env var as floor, returns max(floor, agent level)). Message persistence block in `_make_agent_node` gated by effective level: MINIMAL→role=agent only; STANDARD→user+agent; VERBOSE→all roles. Frontend: `Agent` interface + `AgentCreatePayload` gain `message_log_level` field. `InteractionRulesTab.tsx` gains a Message Log Level dropdown (Inherit/MINIMAL/STANDARD/VERBOSE); save calls `updateInteractionRules` then `updateAgent` sequentially.

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
