# Symphony Program Management — Memory

## Project Structure
- `symph-prgm-mgmt` — umbrella/planning repo (this repo)
- `symph-front-end` — Vite + vanilla HTML/CSS/JS frontend at `~/tech/symphony/symph-front-end`
- `symph-back-end` — FastAPI + LangGraph + PostgreSQL backend at `~/tech/symphony/symph-back-end`

## .claude Directory (read at startup)
- `.claude/agents/symphony-tech-lead.md` — Tech lead agent: decomposes features, defines API contracts, delegates to sub-agents. Works in `symph-prgm-mgmt`. Does NOT write code itself.
- `.claude/agents/symphony-frontend-dev.md` — Frontend agent: vanilla HTML/CSS/JS, Vite, fetch(). Works in `symph-front-end`. Style guide: `doc/agentos.html`.
- `.claude/agents/symphony-backend-dev.md` — Backend agent: FastAPI, LangGraph, PostgreSQL, async SQLAlchemy. Works in `symph-back-end`. Layers: router → service → repository.
- `.claude/skills/implement-feature/SKILL.md` — Multi-agent pipeline skill: Orchestrator → tech-lead → (frontend-dev + backend-dev in parallel) → tech-lead review.
- `.claude/skills/implement-feature/requirements-template.md` — TechRequirementsBlock template used in Step 2 of the implement-feature skill.

## Agent Memory Directories
Each agent has persistent memory at `.claude/agent-memory/<agent-name>/` inside `symph-prgm-mgmt`.
- `.claude/agent-memory/symphony-tech-lead/`
- `.claude/agent-memory/symphony-frontend-dev/`
- `.claude/agent-memory/symphony-backend-dev/`

## Key Conventions
- Frontend: no frameworks (no React/Vue/Angular/jQuery), ES6+, fetch(), textContent over innerHTML
- Backend: async FastAPI, Pydantic models, parameterized SQL, virtualenv named `symphony`
- API base URL: `/api/v1`
- Auth: JWT bearer token via `Depends(get_current_user)`
- Agents never write code for each other's layers
- "Subject Matter Not Known" = the signal when something is unclear (never guess)

## implement-feature Skill Flow
1. Gather inputs (feature_name, feature_description)
2. Produce TechRequirementsBlock
3. Spawn symphony-tech-lead with the block
4. Tech lead designs APIContract, spawns frontend-dev + backend-dev in parallel
5. Tech lead validates integration, returns merged deliverable