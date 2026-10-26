---
key: PER-42
type: story
status: todo
blocks: []
blocked_by: []
labels:
- repo:agent
repos:
- agent
summary: Accept a FastAPI query immediately and publish the PER-20 answer on SSE
---

When FastAPI submits a user question, it needs an immediate ok or rejected reply and a run id, then a single event stream for that run. The agent accepts the run without waiting for the LLM, does the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) answer work in the background, and publishes status on that stream. The 5-second hold applies only to `completed`.

**Depends on.** [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) (`process_question`: direct answer, FastMCP HTTP tool call, synthesis, tool or LLM error). [PER-28](https://linear.app/personalgs/issue/PER-28/fastapi-backend-integration-with-the-agent) is the consumer and stays unchanged.

**Does not change.** Tool names, id normalization, `MCP_URL`, `GET /health`, and the existing blocking `POST /query` that takes `question` and returns `answer` or `error`.

### **Acceptance criteria**

1. **Immediate accept.** Given the agent is up, when `POST /agent/query` receives `{"query": "<non-empty string>"}`, then the response is HTTP 200 `{"status": "ok", "agent_run_id": "<non-empty string>"}` before the LLM or MCP call finishes. The handler does not wait for `process_question`.
2. **Reject without a run.** Given `query` is missing or blank, when `POST /agent/query` is called, then the response is HTTP 200 `{"status": "rejected", "agent_run_id": "<non-empty string>", "reason": "invalid query"}`. The LLM and MCP are not called. `GET /agent/runs/{agent_run_id}/events` sends `failed` with that reason and does not send `completed`.
3. **One event stream.** Given an accepted run, when the caller sends `GET /agent/runs/{agent_run_id}/events` with `Accept: text/event-stream`, then the first event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`. `run_id` in every event is the same id returned by the POST. An unknown id is HTTP 404.
4. **Five-second hold on completed only.** The background job is the existing `process_question` result. When that result is an answer:
   * If the answer is ready less than 5 seconds after `in_progress`, the stream stays on `in_progress` until 5 seconds after that event, then sends `completed` with `data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}` and ends.
   * If the answer is ready at 5 seconds or later, `completed` is sent as soon as the answer is ready and the stream ends.
   * The hold does not delay the POST reply.
5. **Failure is immediate.** When `process_question` returns `{"error": "..."}`, or the LLM or MCP call fails, the stream sends `failed` with `data: {"run_id": "<agent_run_id>", "error": "<that message>"}` as soon as the failure is known. It does not wait out the 5 seconds and it does not send `completed`.
6. [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) **path still works.** `POST /query` with `{"question": "..."}` still returns the blocking `{"answer": "..."}` or `{"error": "..."}`. `GET /health` is unchanged. MCP is still called only from this agent process.

### **Exception criteria**

* FastAPI’s own `run_id` is not an agent field. The agent returns `agent_run_id` and uses that same value as `run_id` inside SSE data. FastAPI forwards those events as sent.
* A second subscriber for the same `agent_run_id` may attach. The agent does not cancel the background job when a subscriber disconnects.
* Auth on these routes stays out of scope, same as [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server).

### **Definition of done**

`POST /agent/query` returns `ok` or `rejected` without waiting for the model. Accepted runs execute the [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) orchestrator in the background. The SSE stream emits `in_progress`, then exactly one of `completed` or `failed`, with the 5-second rule applied only to `completed`. Tests cover immediate return, rejection, the fast and slow completed cases, a tool or LLM failure, an unknown run id, and the existing `POST /query` answer.
