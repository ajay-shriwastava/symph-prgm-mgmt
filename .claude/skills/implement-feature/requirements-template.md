# TechRequirementsBlock Template

Use this template when translating a user's feature description into structured technical
requirements (Step 2 of the implement-feature skill).

Fill every field. Mark genuinely unknown fields `TBD – <reason>` so sub-agents know to
apply sensible defaults. Do not leave fields blank.

---

```
## TechRequirementsBlock

### Meta
- feature_name:        <short identifier, e.g. "Agent Workflow Builder">
- feature_description: <original user description, verbatim>
- tech_stack:
    frontend:  Vanilla HTML + CSS + JavaScript · Vite
    backend:   FastAPI · LangGraph · SQLAlchemy async · PostgreSQL · Alembic · Python 3.11+
- auth_model:          JWT bearer token via FastAPI Depends(get_current_user) | none
- base_api_url:        /api/v1
- involves_langgraph:  yes | no
- involves_websocket:  yes | no
- involves_messaging_channel:  none | telegram | slack | whatsapp

---

### Scope & Acceptance Criteria
"The feature is complete when…"
- [ ] <criterion 1>
- [ ] <criterion 2>

---

### User Roles & Permissions
- <role>: <actions permitted>

---

### Data Entities (PostgreSQL tables via SQLAlchemy)
For each entity:

#### <EntityName>
| Field | Python type | PG column type | Constraints |
|---|---|---|---|
| id | UUID | UUID | PK, default uuid4 |
| name | str | VARCHAR(255) | NOT NULL |
| ... | ... | ... | ... |
| created_at | datetime | TIMESTAMPTZ | server_default now() |
| updated_at | datetime | TIMESTAMPTZ | onupdate now() |

---

### API Surface

#### REST Endpoints
| Method | Path | Auth | Description |
|---|---|---|---|
| GET | /api/v1/<resource> | yes | List … |
| POST | /api/v1/<resource> | yes | Create … |
| GET | /api/v1/<resource>/{id} | yes | Get single … |
| PUT | /api/v1/<resource>/{id} | yes | Update … |
| DELETE | /api/v1/<resource>/{id} | yes | Delete … |

#### WebSocket Channels (if involves_websocket: yes)
| Path | Auth | Purpose |
|---|---|---|
| /ws/<channel> | ?token=<jwt> | <real-time purpose, e.g. live agent logs> |

---

### LangGraph Graph Design (if involves_langgraph: yes)

#### State Schema
| Field | Type | Reducer | Description |
|---|---|---|---|
| messages | List[str] | operator.add | Accumulated messages |
| ... | ... | ... | ... |

#### Graph Topology
```
Nodes:
  <node_name>  — <what it does>

Edges:
  <from>  →  <to>
  <from>  →  <to>  [condition: <expression>]

Entry:  <entry_node>
Exit:   END  (when <condition>)
```

---

### Frontend Screens / Components
- <PageName>: <purpose and key interactions — e.g. "Agent List page: displays all agents, has a Create button that opens a modal form">

---

### Non-Functional Requirements
- Validation:         <key rules, e.g. "name required, max 255 chars; prompt max 4000 chars">
- Error handling:     <expected error states and how they surface in the UI>
- Real-time updates:  <what must update live in the UI, if anything>
- Auth:               JWT stored in localStorage; sent as Authorization: Bearer header
- Performance notes:  <any latency or concurrency notes>
- Accessibility:      WCAG 2.1 AA minimum
```

---

## Guidance for Filling the Template

**Scope & Acceptance Criteria**
Write these as testable statements. "User can create an agent" is bad.
"POST /api/v1/agents with valid body returns 201 and the new agent appears in GET /api/v1/agents" is good.

**LangGraph Graph Design**
Only fill this section if the feature involves running an AI agent graph. Describe the
execution flow (e.g., "router node decides which tool to call → tool node executes →
summariser node formats output → END"). The tech lead will translate this into a
`StateGraph` definition.

**WebSocket Channels**
Only fill this section if the feature needs real-time push (e.g., streaming agent logs,
inter-agent message feed, live token/cost counters). Describe what events flow in each
direction.

**Non-Functional Requirements**
Be explicit about validation (the backend developer will write Pydantic `Field` validators
from these) and about which errors must be surfaced in the UI vs. silently logged.
