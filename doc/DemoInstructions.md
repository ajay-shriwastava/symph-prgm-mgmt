  # 1. Create the database
  psql -U postgres -c "CREATE DATABASE orchestra;"

  # 2. Backend
  cd ~/tech/yuno/orch-back-end && workon orchestra
  pip install -r requirements.txt
  alembic upgrade head
  fastapi dev app/main.py        # → http://127.0.0.1:8000/docs

  # 3. Frontend (new terminal)
  cd ~/tech/yuno/orca-front-end && npm install && npm run dev
  # → http://localhost:5173/src/agents.html

SELECT datname FROM pg_database WHERE datistemplate = false;

SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public';

select * from agents;

select * from workflows;

########
curl -X POST http://127.0.0.1:8000/api/v1/messages \
    -H "Authorization: Bearer test" \
    -H "Content-Type: application/json" \
    -d '{
      "agent_id": "039447ec-ffcc-452a-bba0-3ba384c0a6e7",
      "session_id": "00000000-0000-0000-0000-000000000001",
      "role": "user",
      "content": "Hello, what can you do?"
    }'

########
curl -X POST http://127.0.0.1:8000/api/v1/logs \
    -H "Authorization: Bearer test" \
    -H "Content-Type: application/json" \
    -d '{
      "level": "INFO",
      "message": "Agent started successfully",
      "agent_id": "039447ec-ffcc-452a-bba0-3ba384c0a6e7",
      "metadata": {"duration_ms": 42}
    }'

 How can I test messages and logs                                                                                                                                              
                                    
⏺ Read 2 files (ctrl+o to expand)                                                                                                                                               
                                                                                                                                                                              
⏺ The easiest way is the interactive API docs at http://127.0.0.1:8000/docs. You'll need an agent UUID first.                                                                   
                                                                                                                                                                                
  ---                                                                                                                                                                           
  Step 1 — get an agent UUID                                                                                                                                                    
                                                                                                                                                                                
  If you already created an agent via the UI, grab its ID from the agents page. Or fetch one:                                                                                   
                                                                                                                                                                              
  curl http://127.0.0.1:8000/api/v1/agents \
    -H "Authorization: Bearer test"

  Copy an id from the response.

  ---
  Step 2 — create a message

  curl -X POST http://127.0.0.1:8000/api/v1/messages \
    -H "Authorization: Bearer test" \
    -H "Content-Type: application/json" \
    -d '{
      "agent_id": "<your-agent-uuid>",
      "session_id": "00000000-0000-0000-0000-000000000001",
      "role": "user",
      "content": "Hello, what can you do?"
    }'

  session_id is just a UUID you generate to group a conversation — use any UUID.

  ---
  Step 3 — create a log

  curl -X POST http://127.0.0.1:8000/api/v1/logs \
    -H "Authorization: Bearer test" \
    -H "Content-Type: application/json" \
    -d '{
      "level": "INFO",
      "message": "Agent started successfully",
      "agent_id": "<your-agent-uuid>",
      "metadata": {"duration_ms": 42}
    }'

  agent_id, workflow_id, and metadata are all optional on logs.

  ---
  Step 4 — verify retrieval

  # Messages for your session
  curl "http://127.0.0.1:8000/api/v1/messages?session_id=00000000-0000-0000-0000-000000000001" \
    -H "Authorization: Bearer test"

  # Logs filtered by level
  curl "http://127.0.0.1:8000/api/v1/logs?level=INFO" \
    -H "Authorization: Bearer test"

  The Authorization: Bearer test header works because the auth stub accepts any token.

---

## pgAdmin — Connecting to the Symphony Database

pgAdmin is the recommended GUI for inspecting database state during demos and testing.

### Install pgAdmin (one-time)

Download from https://www.pgadmin.org/download/ and install the macOS `.dmg`, or install via Homebrew:

```bash
brew install --cask pgadmin4
```

### Launch pgAdmin

Open **Finder → Applications → pgAdmin 4** and double-click to launch.

### Register the Symphony server (one-time setup)

1. In the pgAdmin sidebar, right-click **Servers → Register → Server**
2. **General tab** — Name: `Symphony Local`
3. **Connection tab**:
   - Host: `localhost`
   - Port: `5432`
   - Maintenance database: `postgres`
   - Username: `postgres`
   - Password: `postgres` (or your local postgres password)
4. Click **Save**

### Navigate to the Symphony database

In the sidebar, expand:

```
Servers → Symphony Local → Databases → symphony → Schemas → public → Tables
```

Right-click any table → **View/Edit Data → All Rows** to inspect its contents.

### Useful queries (Query Tool)

Open the Query Tool via **Tools → Query Tool**, then run:

```sql
-- List all tables
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public';

-- Agents
SELECT id, name, model, channels, status FROM agents;

-- Workflows and their graph definitions
SELECT id, name, status, graph_definition FROM workflows;

-- Workflow runs
SELECT id, workflow_id, status, started_at, finished_at FROM workflow_runs ORDER BY started_at DESC;

-- Recent messages (including Slack)
SELECT id, agent_id, session_id, role, LEFT(content, 80) AS content, created_at
FROM messages ORDER BY created_at DESC LIMIT 20;

-- Logs
SELECT id, level, message, agent_id, created_at FROM logs ORDER BY created_at DESC LIMIT 20;

-- Reset a stuck workflow run
UPDATE workflow_runs SET status = 'failed', error = 'manually reset' WHERE status = 'running';
```

### Make sure PostgreSQL is running

```bash
brew services list | grep postgresql     # check status
brew services restart postgresql         # restart if needed
```

---

## Visual Workflow Builder — Testing Instructions

### Prerequisites
Both servers must be running:

  # Backend (terminal 1)
  cd ~/tech/yuno/symph-back-end && workon symphony
  fastapi dev        # → http://127.0.0.1:8000

  # Frontend (terminal 2)
  cd ~/tech/yuno/symph-front-end && npm run dev
  # → http://localhost:5173/src/html/workflows.html

### Step 1 — Create a workflow
1. Open http://localhost:5173/src/html/workflows.html
2. Click **+ New Workflow**
3. Enter a name (e.g. `My First Workflow`) and click **Create**
4. The workflow appears in the table

### Step 2 — Open the visual builder
Click **Edit** on the workflow. The SVG canvas builder expands with:
- **Left palette** — Start, Agent, Condition, End node chips
- **Centre canvas** — empty SVG grid
- **Right config panel** — "Select a node" placeholder

### Step 3 — Build a simple Start → End graph
1. Drag **Start** from the palette onto the canvas
2. Drag **End** from the palette to the right of Start
3. Hover the Start node until the purple output port dot appears on the right edge
4. Drag from the output port to the End node's input port (left edge)
5. A curved bezier arrow connects them

### Step 4 — Save
Click **Save** in the toolbar. A "Workflow saved" toast confirms the graph_definition is persisted.

### Step 5 — Run
Click **Run**. The run log panel below the canvas opens and streams live WebSocket events:

  → node_enter: start
  ✓ node_complete: start
  ⟶ edge_traverse: e1
  → node_enter: end
  ✓ node_complete: end
  ✓ Run completed

### Step 6 — Check run history
The **Run History** collapsible section at the bottom lists the run with a `completed` badge and timestamps.

### Step 7 — Test a condition feedback loop
1. Drag **Start → Agent → Condition → End** onto the canvas
2. Click the **Agent** node → pick an agent from the config panel dropdown
3. Click the **Condition** node → set true/false labels
4. Draw edges: Condition `true` port → End, Condition `false` port → Agent (feedback loop)
5. Save and Run — the loop runs up to 5 times then exits automatically via `condition_result = True`

### Step 8 — Verify in pgAdmin
Open pgAdmin (see **pgAdmin — Connecting to the Symphony Database** above) and run:

  SELECT * FROM workflows;        -- graph_definition JSONB, status = 'draft'
  SELECT * FROM workflow_runs;    -- status, output JSONB, started_at, finished_at

---

## LangSmith — Viewing Traces

LangSmith gives you a live view of every LLM call Symphony makes — prompts, responses, token usage, cost, and latency. No code changes are needed; it activates via environment variables.

### Prerequisites

- `LANGCHAIN_TRACING_V2=true`, `LANGCHAIN_API_KEY`, and `LANGCHAIN_PROJECT=symphony` set in `symph-back-end/.env` (see Readme for setup)
- Backend running: `workon symphony && fastapi dev app/main.py`

### Step 1 — Open LangSmith

Go to [smith.langchain.com](https://smith.langchain.com) and sign in.

### Step 2 — Select the Symphony project

In the left sidebar, click **Projects** and select **symphony**. If it doesn't appear yet, trigger a trace first (run a workflow or send a Slack message) — LangSmith creates the project on the first trace.

### Step 3 — View traces

Click the **Traces** tab. Each row is one traced run, showing:
- Run name and status (success / error)
- Model used
- Total tokens and estimated cost
- Latency

### Step 4 — Inspect a trace

Click any row to open the full trace detail:
- **Tree view** — each LangGraph node is a collapsible step; expand to see the exact prompt and response for that node
- **Metadata** — run ID, timestamps, token breakdown (input / output / total)
- **Feedback** — you can manually annotate runs with thumbs up/down for evaluation

### Step 5 — Trigger a workflow trace

1. Open http://localhost:5173/src/html/workflows.html
2. Run any workflow that includes an **Agent** node
3. Switch to LangSmith — the new trace appears within a few seconds

### Step 6 — Trigger a Slack trace

Send a DM or @mention to the Symphony bot in Slack. The resulting LLM call appears in LangSmith as a single-step trace with the agent's system prompt and the user's message.

### Step 7 — Filter and search

Use the **Filter** bar at the top of the Traces list to narrow by:
- **Status**: success / error
- **Model**: e.g. `claude-sonnet-4-6`
- **Latency**: slow runs
- **Date range**: last hour, day, week

---

## Slack Integration — Testing Instructions

### Prerequisites
- A Slack workspace where you can install apps
- `SLACK_BOT_TOKEN` and `SLACK_APP_TOKEN` set in your environment (see Readme)
- Backend running: `workon symphony && fastapi dev app/main.py`

### Step 1 — Configure an agent for Slack
1. Open http://localhost:5173/src/html/memory.html
2. Select an agent from the dropdown
3. Under **Agent Configuration**, find the **Channels** section and add `slack`
4. Save

### Step 2 — Verify the bot connected
Check the backend terminal on startup — you should see:

  INFO  Slack bot connected via Socket Mode.

If you see `SLACK_BOT_TOKEN or SLACK_APP_TOKEN not set — Slack bot disabled`, check your environment variables.

### Step 3 — Send a direct message
1. In Slack, open a DM with your Symphony bot
2. Send any message (e.g. `Hello, what can you do?`)
3. The bot replies using the configured agent's model and system prompt

### Step 4 — Test @mention
1. In any channel where the bot is invited, type `@SymphonyBot Hello`
2. The bot strips the mention prefix and replies

### Step 5 — Verify message persistence
Messages from Slack are stored in the `messages` table and visible in the Symphony UI:

  # Check via SQL
  SELECT * FROM messages ORDER BY created_at DESC LIMIT 10;

  # Or via the UI
  Open http://localhost:5173/src/html/messages.html

### Known behaviour
- The bot routes to the **first** agent with `"slack"` in its channels — only configure one agent per Slack workspace
- The Slack channel ID is used as `session_id`, so each channel maintains its own conversation context
- If no agent has `"slack"` in channels, the bot falls back to `claude-haiku-4-5-20251001` with a generic system prompt (messages are not persisted in this fallback case)
- The bot disables itself silently if tokens are missing or still set to placeholder values

---

### Known behaviour (Workflow Builder)
- Feedback loops exit after **5 agent passes** (MAX_LOOPS = 5) — condition_result is forced to True
- Agent nodes require ANTHROPIC_API_KEY in the environment to call Claude
- Start / End / Condition nodes run without an API key
- Any run stuck in `running` status can be reset:

  UPDATE workflow_runs SET status = 'failed', error = 'manually reset' WHERE status = 'running';