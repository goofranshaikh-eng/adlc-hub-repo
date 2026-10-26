Requirements change. FastAPI must not create, decide, or delay run status.
FastAPI calls the agent POST /query and immediately returns that call's reply, such as Ok or rejected.
The SSE stream only forwards the status and the response the agent already sent. FastAPI does not turn a timer into in_progress, completed, or failed. The 5 second timing stays on the agent, in a later ticket.
FastAPI still does not poll the agent and does not call MCP tools.
