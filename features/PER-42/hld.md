```markdown
# HLD: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Overview

The agent system adds a non-blocking query endpoint (`POST /agent/query`) that accepts user questions immediately, returns a unique `agent_run_id`, and publishes status updates via Server-Sent Events (SSE). The background orchestrator executes the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) work (LLM + MCP integration) without blocking the HTTP response. A 5-second hold applies only to `completed` events; failures are sent immediately.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ FastAPI Application (agent/src/main.py)                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  POST /agent/query              GET /agent/runs/{id}/events
│     │                                    │
│     ├─ Input validation                  ├─ Stream setup
│     ├─ Generate agent_run_id             ├─ in_progress (immediate)
│     ├─ Return 200 immediately            ├─ Publish from queue
│     └─ Launch background job             └─ (completed | failed)
│                                                       ▲
│  ┌─────────────────────────────────────────────────┐ │
│  │ Background Task (src/agent/background_task.py)  │ │
│  │                                                 │ │
│  │  • run_id → agent_run_id                       │ │
│  │  • Execute PER-20 process_question()          │ │
│  │  • on_ready: enqueue_event()                   │ │
│  │  • on_error: enqueue_event()                   │ │
│  │  • Enforce 5-sec hold for completed only      │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │ Run Store (src/agent/run_store.py)              │ │
│  │                                                 │ │
│  │  agent_run_id → [event, event, …]             │─┘
│  │  In-memory dict with asyncio.Queue per run     │
│  │  TTL-based cleanup (1 hour default)            │
│  └─────────────────────────────────────────────────┘
│                                                             │
│  GET /health                POST /query (unchanged)        │
│                                                             │
└─────────────────────────────────────────────────────────────┘

          ▼
┌─────────────────────────────────────────────────────────────┐
│ PER-20 Agent (src/agent/orchestrator.py)                    │
│                                                             │
│  • Takes query → returns answer or error                   │
│  • Calls MCP server via HTTP tool                          │
│  • No changes to signature or behavior                     │
└─────────────────────────────────────────────────────────────┘
```

## Contracts

### 1. POST /agent/query

**Request:**
```json
POST /agent/query
Content-Type: application/json

{
  "query": "<non-empty string>"
}
```

**Responses:**

**Success (AC-1):**
```json
HTTP 200
Content-Type: application/json

{
  "status": "ok",
  "agent_run_id": "<uuid v4 hex>"
}
```

**Rejection (AC-2):**
```json
HTTP 200
Content-Type: application/json

{
  "status": "rejected",
  "agent_run_id": "<uuid v4 hex>",
  "reason": "invalid query"
}
```

**Behavior:**
- Input validation: reject if `query` is missing, `None`, or empty string (whitespace-only counts as invalid).
- Return HTTP 200 immediately; do not wait for `process_question` or any LLM/MCP call.
- Generate and return `agent_run_id` in both success and rejection cases.
- On rejection: do not launch background job; do not call LLM or MCP. Enqueue `in_progress` and `failed` events synchronously with reason `"invalid query"` before POST returns.
- On acceptance: launch background job asynchronously via `asyncio.create_task()` without awaiting; enqueue `in_progress` event synchronously before POST returns.

### 2. GET /agent/runs/{agent_run_id}/events

**Request:**
```http
GET /agent/runs/{agent_run_id}/events
Accept: text/event-stream
```

**Response Headers:**
```http
HTTP 200
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

**Event Stream Format (SSE):**

Each event is a JSON object sent as `data: <json>`.

**Event 1: in_progress (AC-3)**
```
event: in_progress
data: {"run_id": "<agent_run_id>"}
```
Sent immediately when the run is accepted (enqueued before POST returns), before any background work begins.

**Event 2a: completed (AC-4)**
```
event: completed
data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
```
Sent when `process_question` returns a successful answer. Timing:
- If answer is ready **less than 5 seconds** after `in_progress` was enqueued: hold until 5 seconds have elapsed, then send `completed`.
- If answer is ready **5 seconds or later** after `in_progress`: send `completed` immediately (no extra hold).

**Event 2b: failed (AC-5)**
```
event: failed
data: {"run_id": "<agent_run_id>", "error": "<error message>"}
```
Sent immediately (no 5-second hold) when `process_question` returns `{"error": "…"}` or when LLM/MCP call fails, or when the query is rejected in the POST handler (from AC-2 rejection path). Stream ends after this event.

**Error Response:**
```json
HTTP 404
Content-Type: application/json

{
  "detail": "Run not found"
}
```
Sent if `agent_run_id` does not exist or has been cleaned up by TTL expiration.

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream ends after `completed` or `failed`.
- Every event contains the same `run_id` (the `agent_run_id` from POST).
- Multiple subscribers for the same `agent_run_id` are allowed; the background job is not cancelled when a subscriber disconnects.

### 3. POST /query (unchanged)

**Request:**
```json
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response:**
```json
HTTP 200
Content-Type: application/json

{
  "answer": "<text>"
}
```
or
```json
{
  "error": "<error message>"
}
```

**Behavior:** Blocking; calls `process_question` directly and waits for result. No changes.

### 4. GET /health (unchanged)

**Response:**
```json
HTTP 200
Content-Type: application/json

{"status": "ok"}
```

**Behavior:** No changes.

---

## Data Flow

### Acceptance Path (AC-1)

1. **POST /agent/query** receives `{"query": "What is …?"}`
2. Handler validates input (custom handler catches empty/whitespace after strip).
3. Handler generates `agent_run_id` (UUID v4 hex).
4. Handler creates run in store; `in_progress` event enqueued synchronously.
5. Handler spawns background task via `asyncio.create_task(run_background_job(agent_run_id, query))` **without awaiting**.
6. Handler returns HTTP 200 `{"status": "ok", "agent_run_id": "…"}` **immediately** (before background task executes).
7. **GET /agent/runs/{agent_run_id}/events** opens SSE stream.
8. Stream reads from event queue; delivers `in_progress` event (already queued before POST returned).
9. Background task (running asynchronously) calls `process_question(query)` from PER-20.
10. On success: enqueue `completed` event after enforcing 5-second hold (see AC-4 timing).
11. On error: enqueue `failed` event immediately (no hold).
12. Stream reader picks up result event and sends to subscriber(s).
13. Stream closes after `completed` or `failed`.

### Rejection Path (AC-2)

1. **POST /agent/query** receives `{"query": ""}` or missing/None.
2. Handler validates input (custom validation catches missing/empty/whitespace).
3. Handler generates `agent_run_id` (UUID v4 hex).
4. Handler creates run in store; `in_progress` event enqueued synchronously.
5. Handler marks run failed; `failed` event enqueued synchronously with reason `"invalid query"`.
6. Handler returns HTTP 200 `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}` **immediately**.
7. **Do NOT spawn background job; do not call `process_question`.**
8. **GET /agent/runs/{agent_run_id}/events** opens SSE stream.
9. Stream reads from event queue; delivers `in_progress` event, then `failed` event (both queued before POST returned).
10. No `completed` event sent.
11. Stream closes after `failed`.

### 5-Second Hold Enforcement (AC-4)

```
Timeline for fast answer (< 5 sec):
  t=0.0:  in_progress enqueued and timestamp recorded (in_progress_sent_at)
  t=0.5:  process_question returns answer
  t=0.5:  elapsed = 0.5s; hold_remaining = 5.0 - 0.5 = 4.5s
  t=0.5:  await asyncio.sleep(4.5) starts in background task
  t=5.0:  sleep completes; completed event enqueued
  
Timeline for slow answer (>= 5 sec):
  t=0.0:  in_progress enqueued and timestamp recorded
  t=6.0:  process_question returns answer
  t=6.0:  elapsed = 6.0s; hold_remaining = 5.0 - 6.0 = -1.0s
  t=6.0:  hold_remaining <= 0; send completed immediately (no sleep)
```

---

## Integration with PER-20

**Dependency:** This feature depends on [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) (`process_question`), which:
- Takes a `query` string and returns `{"answer": "<text>"}` or `{"error": "<msg>"}`.
- Calls the MCP server via HTTP tool.
- Handles LLM synthesis and error cases.
- Is called from the main agent process only.

**Changes to PER-20:** None. The `process_question` function's signature and behavior remain unchanged. This feature wraps it in a non-blocking async handler (via `run_background_job` in `agent/src/agent/background_task.py`).

**Consumer:** [PER-28](https://linear.app/personalgs/issue/PER-28/fastapi-backend-integration-with-the-agent) consumes SSE events as published; no changes needed.

---

## Key Invariants

1. **No wait on POST:** HTTP 200 is returned before `process_question` is called; response sent immediately (within ~1ms after event enqueue).
2. **One stream per run:** A single SSE stream publishes all events for a run; multiple subscribers are allowed and see identical events.
3. **Event sequence:** `in_progress` → exactly one of (`completed` | `failed`) → stream closes.
4. **5-second hold is local to background task:** Only `completed` is held; `failed` is sent immediately; the POST response is never delayed by hold.
5. **Rejection never launches a background job:** Invalid input does not call LLM or MCP. Events (`in_progress`, `failed`) are enqueued synchronously before POST returns; no background task spawned.
6. **Backward compatibility:** `POST /query`, `GET /health`, tool names, and MCP URL are unchanged.

---

## Implementation Notes

### File Locations (per change_requests.md)

All routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query, GET /health) are registered in **`agent/src/main.py`** only. No separate `agent/src/routes/agent.py` or `agent/src/agent/routes.py` file.

### Run Store

Each run identified by `agent_run_id` (UUID v4 hex string):

```python
run_store[agent_run_id] = {
  "id": "<agent_run_id>",
  "query": "<user query>",
  "created_at": <unix timestamp>,
  "status": "pending" | "completed" | "failed",
  "events": asyncio.Queue([
    {"event": "in_progress", "data": {"run_id": "<agent_run_id>"}},
    {"event": "completed", "data": {"run_id": "<agent_run_id>", "response": {"answer": "…"}}}
      # or
    {"event": "failed", "data": {"run_id": "<agent_run_id>", "error": "…"}},
  ]),
  "result": None | {"answer": "…"} | {"error": "…"},
  "started_at": <unix timestamp> | None,
  "ended_at": <unix timestamp> | None,
  "in_progress_sent_at": <unix timestamp>
}
```

### Pydantic Models

```python
# Request validation (agent/src/agent/models.py)
class AgentQueryRequest(BaseModel):
  query: str = Field(..., min_length=1)

# Responses
class AgentQueryAccepted(BaseModel):
  status: Literal["ok"]
  agent_run_id: str

class AgentQueryRejected(BaseModel):
  status: Literal["rejected"]
  agent_run_id: str
  reason: str

# SSE event data
class InProgressEventData(BaseModel):
  run_id: str

class CompletedEventData(BaseModel):
  run_id: str
  response: dict  # {"answer": str}

class FailedEventData(BaseModel):
  run_id: str
  error: str
```

### Input Validation Handling

FastAPI's automatic Pydantic validation will reject missing/non-string `query` at HTTP 422. A custom exception handler must convert 422 validation errors to HTTP 200 `{"status": "rejected", "agent_run_id": "<id>", "reason": "invalid query"}` responses (implementation detail in `agent/src/main.py` POST /agent/query handler or middleware).

---

## Implementation Risks & Mitigations

| Risk | Impact | Severity | Mitigation |
|------|--------|----------|-----------|
| **Event loss if process crashes before SSE connects** | Subscriber misses `in_progress` or answers. | Medium | Queue events in in-memory `asyncio.Queue` per run before SSE stream reads. Replay on stream open. TTL-based cleanup deletes old runs. |
| **Unbounded run accumulation** | Memory leak or OOM crash. | High | Implement TTL-based cleanup; delete runs after 1 hour by default. Log cleanup policy. Configurable via `RUN_CLEANUP_TTL_SECONDS` env var. |
| **Multiple subscribers desync** | One subscriber slow; others see different timings. | Low | All subscribers read from same event queue in order; no per-subscriber logic. All see identical timing and content. |
| **5-second hold blocks other work** | System throughput reduced if hold is blocking. | Low | Hold enforced via `asyncio.sleep()` in background task (non-blocking), not in event stream reader. Does not block other runs. |
| **Unknown run_id returns 404 immediately** | Caller assumes run failed, but job may still be running. | Low | Clarify API docs: 404 means run never created or TTL expired. Keep runs ≥ 1 hour. Configurable via env var. |
| **PER-20 contract changes** | Breaking change to new endpoint. | High | Design depends on stable `process_question(query: str) → {"answer": "…"} \| {"error": "…"}` return type. Lock contract in AC-6 tests. |
| **Validation layer returns 422 instead of 200** | Violates AC-1, AC-2 requirement for HTTP 200 responses. | Medium | Custom exception handler converts 422 to 200 rejected response before handler body executes. |

---

## Story Breakdown

This epic is decomposed into three stories, executed in order:

1. **Story PER-43 (01-agent): Accept FastAPI query without blocking; return agent_run_id immediately** (AC-1, AC-2)
   - Acceptance Criteria: AC-1, AC-2
   - Test Count: 8
   - Key deliverable: POST /agent/query endpoint with immediate validation and response, run store initialization, background job spawn via `asyncio.create_task()`, input validation with custom exception handler for 422→200 conversion.
   - Implementations: `agent/src/agent/models.py`, `agent/src/agent/run_store.py`, `agent/src/agent/background_task.py` (spawn function only), `agent/src/main.py` (POST /agent/query handler).
   - See `stories/01-agent.md` for detailed LLD.

2. **Story PER-44 (02-agent): Stream PER-20 answer via SSE with in_progress, completed, or failed events** (AC-3, AC-4, AC-5)
   - Acceptance Criteria: AC-3, AC-4, AC-5
   - Test Count: 15
   - Key deliverable: GET /agent/runs/{agent_run_id}/events endpoint with SSE delivery, background task orchestration (call `process_question`, enforce 5-second hold), event enqueue to run store, TTL-based cleanup task startup.
   - Implementations: `agent/src/agent/sse_handler.py`, `agent/src/agent/background_task.py` (full orchestrator), `agent/src/agent/run_store.py` (extend with cleanup), `agent/src/main.py` (GET endpoint and cleanup task).
   - See `stories/02-agent.md` for detailed LLD.

3. **Story PER-45 (03-agent): Preserve existing POST /query and GET /health endpoints** (AC-6)
   - Acceptance Criteria: AC-6
   - Test Count: 9
   - Key deliverable: Audit and regression tests to confirm no interference with existing endpoints, MCP process isolation, tool names and configuration unchanged.
   - No code changes; audit only.
   - See `stories/03-agent.md` for detailed LLD.

**Total: 3 stories, 32 tests, covering all 6 acceptance criteria.**

---

## Acceptance Criteria Coverage

| AC | Story | Test Count | Coverage |
|----|-------|------------|----------|
| AC-1: Immediate accept | PER-43 (01-agent) | 2 | Valid query returns ok with agent_run_id immediately (< 100ms); response sent before process_question called |
| AC-2: Reject without a run | PER-43 (01-agent) | 6 | Empty/missing/whitespace query returns rejected with reason; no LLM/MCP called; failed event enqueued synchronously |
| AC-3: One event stream | PER-44 (02-agent) | 7 | HTTP 200 text/event-stream; first event in_progress; all events same run_id; 404 on unknown; multi-subscriber support; event replay on late subscription |
| AC-4: Five-second hold | PER-44 (02-agent) | 4 | Fast answer (< 5s) held until 5s elapsed; slow answer (≥ 5s) sent immediately; edge case at exactly 5s; stream closes after completed |
| AC-5: Failure immediate | PER-44 (02-agent) | 3 | LLM/MCP errors send failed immediately; invalid query sends failed; stream closes after failed (no completed event) |
| AC-6: Existing paths unchanged | PER-45 (03-agent) | 9 | POST /query blocking behavior and latency unchanged; GET /health unchanged; coexistence of both endpoints; MCP process isolation; tool names and MCP_URL unchanged |
| **Total** | **3 stories** | **32 tests** | **All 6 ACs covered** |

---

## Testing Summary

- **Unit tests:** 32 tests across 3 stories verify all acceptance criteria.
- **Test categories:** Input validation, immediate response, background execution, SSE delivery, event timing, failure handling, multi-subscriber support, backward compatibility.
- **Integration test:** Full flow from POST /agent/query through background orchestration to SSE completion, plus rejection flow and legacy endpoint coexistence.

All tests are documented in the respective story LLDs:
- `stories/01-agent.md` — Story 1 tests (AC-1, AC-2)
- `stories/02-agent.md` — Story 2 tests (AC-3, AC-4, AC-5)
- `stories/03-agent.md` — Story 3 tests (AC-6)

---

## Decisions & Deferred Items

### Adopted

- **Event queue:** In-memory `asyncio.Queue` per run (proof-of-concept); upgrade to Redis if multi-process deployment required.
- **5-second hold:** Enforced via `asyncio.sleep()` in background task; does not block other runs or the event stream.
- **Run cleanup:** TTL-based; runs deleted 1 hour after completion or failure by default (configurable via `RUN_CLEANUP_TTL_SECONDS` env var).
- **Run ID format:** UUID v4 hex (32-character string, collision-free).
- **Input validation:** O(1) string check (strip whitespace, test empty); synchronous before task spawn.
- **Validation layer:** Custom exception handler converts HTTP 422 (Pydantic validation error) to HTTP 200 rejected response per AC-1/AC-2 contract.
- **Routes location:** All routes (POST /agent/query, GET /agent/runs/{id}/events, POST /query, GET /health) in `agent/src/main.py` only (per change_requests.md).

### Deferred

- **Auth/authorization:** Out of scope per requirements; same as PER-20.
- **Rate limiting:** No per-user or global throttle; assume upstream FastAPI middleware.
- **Metrics/logging:** No instrumentation of run duration, latency, or error rates beyond startup logs; out of scope.
- **Run history API:** No `GET /agent/runs` or `GET /agent/runs/{id}` metadata endpoint; only event stream (`GET /agent/runs/{id}/events`).
- **Persistent event store:** In-memory queue is PoC; upgrade to Redis if > 100 concurrent runs or multi-process deployment required.

---

## Risks & Open Questions

| Item | Answer |
|------|--------|
| **Max queue depth?** | Unconstrained in-memory queue; upgrade to Redis if > 100 concurrent runs. |
| **PER-20 changes?** | Design depends on current `process_question` contract. Any change must preserve return type: `{"answer": "…"}` or `{"error": "…"}`. |
| **Run history?** | Deferred; query-only reads via SSE. No metadata endpoint in this epic. |
| **Subscriber disconnect?** | Background job continues; no cancellation. New subscribers retrieve all events (no replay loss). |
| **Configurable hold duration?** | Yes; expose as `AGENT_COMPLETE_HOLD_SECONDS` env var (default 5 seconds). |
| **Cleanup interval?** | Configurable via `RUN_CLEANUP_INTERVAL_SECONDS` env var (default 60 seconds). |
```
