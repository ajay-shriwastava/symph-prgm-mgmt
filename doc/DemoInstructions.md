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
      "agent_id": "88344ceb-c0c4-453b-990e-86258a89a106",
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
      "agent_id": "88344ceb-c0c4-453b-990e-86258a89a106",
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