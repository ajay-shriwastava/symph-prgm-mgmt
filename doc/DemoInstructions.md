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
Connect to the `symphony` database and check:

  SELECT * FROM workflows;        -- graph_definition JSONB, status = 'draft'
  SELECT * FROM workflow_runs;    -- status, output JSONB, started_at, finished_at

### Known behaviour
- Feedback loops exit after **5 agent passes** (MAX_LOOPS = 5) — condition_result is forced to True
- Agent nodes require ANTHROPIC_API_KEY in the environment to call Claude
- Start / End / Condition nodes run without an API key
- Any run stuck in `running` status can be reset:

  UPDATE workflow_runs SET status = 'failed', error = 'manually reset' WHERE status = 'running';