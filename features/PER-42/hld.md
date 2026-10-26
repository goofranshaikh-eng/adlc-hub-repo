```markdown
# High-Level Design: Accept FastAPI query immediately and publish PER-20 answer on SSE

## Overview

The agent system adds a non-blocking query endpoint (`POST /agent/query`) that accepts user questions immediately, returns a unique `agent_run_id`, and publishes status updates via Server-Sent Events (SSE). The background orchestrator executes the existing [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) work (LLM + MCP integration) without blocking the HTTP response. A 5-second hold applies only to `completed` events; failures are sent immediately.

## Component Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ FastAPI Application                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  POST /agent/query                 GET /agent/runs/{id}/events
│     │                                       │
│     ├─ Input validation                     ├─ Stream setup
│     ├─ Generate agent_run_id                ├─ in_progress (immediate)
│     ├─ Return 200 immediately               ├─ Publish from queue
│     └─ Launch background job                └─ (completed | failed)
│                                                       ▲
│  ┌─────────────────────────────────────────────────┐ │
│  │ Background Task Queue                          │ │
│  │                                                 │ │
│  │  • run_id → agent_run_id                       │ │
│  │  • Execute PER-20 process_question()          │ │
│  │  • on_ready: enqueue event_message             │ │
│  │  • on_error: enqueue error_message             │ │
│  │  • Enforce 5-sec hold for completed only      │ │
│  └─────────────────────────────────────────────────┘ │
│                                                       │
│  ┌─────────────────────────────────────────────────┐ │
│  │ Run Event Store (in-memory or Redis)           │ │
│  │                                                 │ │
│  │  agent_run_id → [event, event, …]             │─┘
│  │  Subscribers connected per run_id              │
│  └─────────────────────────────────────────────────┘
│                                                             │
│  GET /health                    POST /query (unchanged)    │
│                                                             │
└─────────────────────────────────────────────────────────────┘

          ▼
┌─────────────────────────────────────────────────────────────┐
│ PER-20 Agent (process_question)                             │
│                                                             │
│  • Takes question → returns answer or error                │
│  • Calls MCP server via HTTP tool                          │
│  • No changes to signature or behavior                     │
└─────────────────────────────────────────────────────────────┘
```

## Contracts

### 1. POST /agent/query

**Request:**
```
POST /agent/query
Content-Type: application/json

{
  "query": "<non-empty string>"
}
```

**Responses:**

**Success (AC-1):**
```
HTTP 200
Content-Type: application/json

{
  "status": "ok",
  "agent_run_id": "<uuid or sequential id>"
}
```

**Rejection (AC-2):**
```
HTTP 200
Content-Type: application/json

{
  "status": "rejected",
  "agent_run_id": "<uuid or sequential id>",
  "reason": "invalid query"
}
```

**Behavior:**
- Input validation: reject if `query` is missing, `None`, or empty string (whitespace-only counts as invalid).
- Return HTTP 200 immediately; do not wait for `process_question` or any LLM/MCP call.
- Generate and return `agent_run_id` in both success and rejection cases.
- On rejection: do not launch background job; do not call LLM or MCP. Enqueue `failed` event synchronously with reason `"invalid query"`.
- On acceptance: launch background job asynchronously; enqueue `in_progress` event immediately.

### 2. GET /agent/runs/{agent_run_id}/events

**Request:**
```
GET /agent/runs/{agent_run_id}/events
Accept: text/event-stream
```

**Response Headers:**
```
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
Sent immediately when the run is accepted, before any background work begins.

**Event 2a: completed (AC-4)**
```
event: completed
data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}
```
Sent when `process_question` returns a successful answer. Timing:
- If answer is ready **less than 5 seconds** after `in_progress` is sent: hold until 5 seconds have elapsed, then send `completed`.
- If answer is ready **5 seconds or later** after `in_progress`: send `completed` immediately.

**Event 2b: failed (AC-5)**
```
event: failed
data: {"run_id": "<agent_run_id>", "error": "<error message>"}
```
Sent immediately (no 5-second hold) when `process_question` returns `{"error": "…"}` or when LLM/MCP call fails, or when the query is rejected in the POST handler. Stream ends after this event.

**Error Response:**
```
HTTP 404
Content-Type: application/json

{
  "detail": "Run not found"
}
```
Sent if `agent_run_id` does not exist or has already been cleaned up.

**Behavior:**
- First event is always `in_progress`.
- Exactly one of `completed` or `failed` follows.
- Stream ends after `completed` or `failed`.
- Every event contains the same `run_id` (the `agent_run_id` from POST).
- Multiple subscribers for the same `agent_run_id` are allowed; the background job is not cancelled when a subscriber disconnects.

### 3. POST /query (unchanged)

**Request:**
```
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response:**
```
HTTP 200
Content-Type: application/json

{
  "answer": "<text>"
}
```
or
```
{
  "error": "<error message>"
}
```

**Behavior:** Blocking; calls `process_question` directly and waits for result. No changes.

### 4. GET /health (unchanged)

**Response:**
```
HTTP 200
Content-Type: application/json

{"status": "ok"}
```

**Behavior:** No changes.

---

## Data Flow

### Acceptance Path (AC-1)

1. **POST /agent/query** receives `{"query": "What is …?"}`
2. Handler validates input.
3. Handler generates `agent_run_id` (UUID or sequential).
4. Handler enqueues `in_progress` event with that `run_id`.
5. Handler spawns background task with `(agent_run_id, query)`.
6. Handler returns HTTP 200 `{"status": "ok", "agent_run_id": "…"}` **immediately**.
7. **GET /agent/runs/{agent_run_id}/events** opens SSE stream.
8. Stream delivers `in_progress` event (already queued).
9. Background task calls `process_question(query)`.
10. On success: enqueue `completed` event after enforcing 5-second hold (see AC-4 timing).
11. On error: enqueue `failed` event immediately.
12. Stream reads events and sends them to subscriber(s).
13. Stream closes after `completed` or `failed`.

### Rejection Path (AC-2)

1. **POST /agent/query** receives `{"query": ""}` or `{"query": None}`.
2. Handler validates input; validation fails.
3. Handler generates `agent_run_id`.
4. Handler enqueues `in_progress` event.
5. Handler enqueues `failed` event with reason `"invalid query"`.
6. Handler returns HTTP 200 `{"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}` **immediately**.
7. **GET /agent/runs/{agent_run_id}/events** opens SSE stream.
8. Stream delivers `in_progress` event, then `failed` event (both already queued); no `completed`.
9. Stream closes.

### 5-Second Hold Enforcement (AC-4)

```
Timeline for fast answer (< 5 sec):
  t=0.0:  in_progress sent to stream
  t=0.5:  process_question returns answer
  t=0.5:  hold_until = t + 5.0 = t=5.5
  t=5.5:  now >= hold_until; send completed
  
Timeline for slow answer (>= 5 sec):
  t=0.0:  in_progress sent to stream
  t=6.0:  process_question returns answer
  t=6.0:  hold_until = max(t, 5.0) = t=6.0 (already elapsed)
  t=6.0:  send completed immediately
```

---

## Integration with PER-20

**Dependency:** This feature depends on [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) (`process_question`), which:
- Takes a `query` string and returns `{"answer": "<text>"}` or `{"error": "<msg>"}`.
- Calls the MCP server via HTTP tool.
- Handles LLM synthesis and error cases.
- Is called from the main agent process only.

**Changes to PER-20:** None. The `process_question` function's signature and behavior remain unchanged. This feature wraps it in a non-blocking async handler.

**Consumer:** [PER-28](https://linear.app/personalgs/issue/PER-28/fastapi-backend-integration-with-the-agent) consumes SSE events as published; no changes needed.

---

## Key Invariants

1. **No wait on POST:** HTTP 200 is returned before `process_question` is called or completes.
2. **One stream per run:** A single SSE stream publishes all events for a run; multiple subscribers are allowed.
3. **Event sequence:** `in_progress` → exactly one of (`completed` | `failed`) → stream closes.
4. **5-second hold is local:** Only `completed` is held; `failed` is sent immediately; the POST response is never delayed.
5. **Rejection never launches a background orchestrator job:** Invalid input does not call LLM or MCP. Events (`in_progress`, `failed`) are enqueued synchronously before POST returns.
6. **Backward compatibility:** `POST /query`, `GET /health`, tool names, and MCP URL are unchanged.

---

## Implementation Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|-----------|
| **Event loss if process crashes before SSE connects** | Subscriber misses `in_progress` or answers. | Queue events in durable store (in-memory asyncio.Queue for PoC, or Redis for multi-process) before SSE stream reads them. Query and replay on stream open. |
| **Unbounded run accumulation** | Memory leak or disk fill. | Implement TTL-based cleanup (e.g., delete runs after 1 hour or on explicit purge). Log cleanup policy. |
| **Multiple subscribers desync** | One subscriber slow; others see different timings. | All events are read from the same queue in order; no per-subscriber logic. All subscribers see identical timing. |
| **5-second hold on fast answers blocks other work** | System throughput reduced if hold is blocking. | Implement hold as a scheduled async task, not a sleep. Do not block the event queue or LLM calls. |
| **Unknown run_id returns 404 immediately** | Caller assumes run failed, but job may still be running. | Clarify in API docs: 404 means run was never created or has been cleaned up. Return 404 only for truly unknown ids; keep runs for at least 1 hour (configurable). |
| **PER-20 signature changes** | Breaking change to new endpoint. | This feature depends on the current PER-20 contract. Any future change to `process_question` must preserve the return type. |

---

## Story Breakdown

See `breakdown.md` for the complete breakdown of this epic into three stories:

1. **Story 1: Accept FastAPI query without blocking; return agent_run_id immediately** (AC-1, AC-2)
   - 8 verification tests
   - File paths: see `stories/01-agent.md` and `specs/1.md`

2. **Story 2: Stream PER-20 answer via SSE with in_progress, completed, or failed events** (AC-3, AC-4, AC-5)
   - 15 verification tests
   - File paths: see `stories/02-agent.md` and `specs/2.md`

3. **Story 3: Preserve existing POST /query and GET /health endpoints** (AC-6)
   - 9 verification tests
   - File paths: see `stories/03-agent.md` and `specs/3.md`

**Total: 3 stories, 32 tests, covering all 6 acceptance criteria.**

---

## Schema & API Notes

### Run Store

Each run is identified by a unique `agent_run_id` (string: UUID v4 hex).

```
run_store[agent_run_id] = {
  "id": "<agent_run_id>",
  "query": "<user query>",
  "created_at": <unix timestamp>,
  "status": "pending" | "completed" | "failed",
  "events": [
    {"event": "in_progress", "data": {"run_id": "<agent_run_id>"}},
    {"event": "completed", "data": {"run_id": "<agent_run_id>", "response": {"answer": "…"}}},
    // or
    {"event": "failed", "data": {"run_id": "<agent_run_id>", "error": "…"}},
  ],
  "result": null | {"answer": "…"} | {"error": "…"},
  "started_at": <unix timestamp> | null,
  "ended_at": <unix timestamp> | null,
  "in_progress_sent_at": <unix timestamp>
}
```

### Models (FastAPI/Pydantic)

```
# Request
class AgentQueryRequest:
  query: str

# Response (success)
class AgentQueryAccepted:
  status: Literal["ok"]
  agent_run_id: str

# Response (rejection)
class AgentQueryRejected:
  status: Literal["rejected"]
  agent_run_id: str
  reason: str

# SSE Event data
class InProgressEventData:
  run_id: str

class CompletedEventData:
  run_id: str
  response: {"answer": str}

class FailedEventData:
  run_id: str
  error: str
```

---

## Decisions & Deferred Items

### Adopted

- **Event queue:** In-memory (Python `asyncio.Queue`) per run for proof-of-concept; upgrade to Redis if multi-process deployment is required.
- **5-second hold:** Enforced via async timer per run in background task; does not block other runs or the event stream.
- **Run cleanup:** TTL-based; runs deleted 1 hour after completion or failure (configurable via `RUN_CLEANUP_TTL_SECONDS` env var).
- **Run ID format:** UUID v4 hex (32-character string, collision-free).

### Deferred

- **Auth:** Out of scope per requirements; same as PER-20.
- **Rate limiting:** No per-user or global throttle; assume upstream FastAPI rate limiting.
- **Metrics:** No logging of run duration, answer latency, or error rates; out of scope.
- **Run history API:** No `GET /agent/runs` or `GET /agent/runs/{id}` for metadata; only event stream.

---

## Acceptance Criteria Coverage

| AC | Story | Tests | Coverage |
|----|-------|-------|----------|
| AC-1: Immediate accept | Story 1 | 2 | Valid query returns ok with agent_run_id immediately (< 100ms) |
| AC-2: Reject without a run | Story 1 | 6 | Empty/missing/whitespace query returns rejected; no LLM/MCP called; `failed` event enqueued |
| AC-3: One event stream | Story 2 | 7 | HTTP 200 with text/event-stream; first event in_progress; all events same run_id; 404 on unknown; multi-subscriber support; event replay |
| AC-4: Five-second hold | Story 2 | 4 | Fast answer held until 5s; slow answer immediate; edge case at 5s; stream closes after completed |
| AC-5: Failure immediate | Story 2 | 3 | LLM/MCP errors send failed immediately; invalid query sends failed; stream closes after failed |
| AC-6: Existing paths unchanged | Story 3 | 9 | POST /query blocking behavior and latency unchanged; GET /health unchanged; coexistence; MCP process isolation; tool names and MCP_URL unchanged |
| **Total** | **3 stories** | **32 tests** | **All 6 ACs covered** |

---

## Testing Summary

- **Unit tests:** 32 tests across 3 stories verify all acceptance criteria.
- **Test categories:** Input validation, immediate response, background execution, SSE delivery, event timing, failure handling, multi-subscriber support, backward compatibility.
- **Integration test:** Full flow from POST /agent/query through background orchestration to SSE completion, plus rejection flow and legacy endpoint coexistence.

---

## Risks & Open Questions

| Question | Answer |
|----------|--------|
| **What is the max queue depth?** | Unconstrained in-memory queue; upgrade to Redis if > 100 concurrent runs. |
| **What if PER-20 changes?** | This design depends on the current `process_question` contract. Any change must preserve the return type. |
| **Should we store run history?** | Deferred; query-only reads are supported via SSE; no metadata endpoint. |
| **What if a subscriber disconnects mid-stream?** | The background job continues; no cancellation. New subscribers can connect and retrieve all events (no replay loss). |
| **Should 5-second hold be configurable?** | Yes, as an environment variable or config file; default 5 seconds (`AGENT_COMPLETE_HOLD_SECONDS`). |
| **What file paths should be used?** | See `breakdown.md` and story files (`stories/01-agent.md`, `stories/02-agent.md`, `stories/03-agent.md`) and tech stubs (`specs/1.md`, `specs/2.md`, `specs/3.md`) for concrete repo-specific paths. |
```
