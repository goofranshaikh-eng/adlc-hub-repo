---
key: PER-28
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:fastapi
repos:
- fastapi
summary: fastapi-backend integration with the agent
---

POST /query returns immediately with whatever the agent sends on that call, such as Ok or rejected. FastAPI returns that same immediate reply to its caller. This reply is not delayed, and it is not the agent's finished answer.

The caller reads one SSE stream for that run. The stream carries the status and, when the run finishes, the agent's response.

SSE events:

* The first event is in_progress, sent when the run starts.
* If the agent's response is ready in under 5 seconds, the stream stays at in_progress until 5 seconds after that first event, then sends completed with the response and ends.
* If the agent's response takes 5 seconds or longer, the stream sends completed with the response as soon as it is ready, and ends.
* If the run fails, the stream sends failed as soon as the failure is known, does not send completed, and ends.

The 5 second wait delays only the completed event. It does not delay the immediate Ok or rejected reply, and it does not hold the agent call.

FastAPI does not poll the agent for status or for the response. FastAPI does not call MCP tools.
