```markdown
# Story 01-agent: Accept FastAPI query without blocking; return agent_run_id immediately

**Repository:** `agent`  
**Acceptance Criteria:** AC-1, AC-2  
**Story Type:** Backend Feature  

---

## Acceptance Criteria

### AC-1: Immediate accept

**Given** the agent is up and running  
**When** `POST /agent/query` receives `{"query": "<non-empty string>"}`  
**Then** the response is HTTP 200 `{"status": "ok", "agent_run_id": "<non-empty string>"}` before the LLM or MCP call finishes, and the handler does not wait for `process_question` to complete.

### AC-2: Reject without a run

**Given** `query` is missing or blank  
**When** `POST /agent/query` is called  
**Then** the response is HTTP 200 `{"status": "rejected", "agent_run_id": "<non-empty string>", "reason": "invalid query"}`, the LLM and MCP are not called, and `GET /agent/runs/{agent_run_id}/events` sends `failed` with that reason and does not send `completed`.

---

## File Plan

All paths resolve from the `agent` repository root.

### New Files

```
agent/
├── src/
│   ├── agent/
│   │   ├── __init__.py
│   │   ├── models.py
│   │   ├── run_store.py
│   │   ├── orchestrator.py
│   │   └── routes.py
│   └── main.py
└── tests/
    └── test_agent_query_accept.py
```

**Key files:**

1. **`agent/src/agent/models.py`** — Pydantic models for request/response validation.
2. **`agent/src/agent/run_store.py`** — In-memory run registry; manages `agent_run_id` and event queues.
3. **`agent/src/agent/orchestrator.py`** — Background task spawn logic; interface to `process_question`.
4. **`agent/src/agent/routes.py`** — FastAPI route handler for `POST /agent/query`.
5. **`agent/src/main.py`** — Application startup; register routes and run store.
6. **`agent/tests/test_agent_query_accept.py`** — Unit tests for AC-1 and AC-2.

### Modified Files

```
agent/
├── src/requirements.txt (verify FastAPI, uvicorn, pydantic, httpx present)
└── README.md (document new endpoint and agent_run_id concept)
```

---

## Interfaces

### Request Model (`agent/src/agent/models.py`)

```python
from pydantic import BaseModel, Field

class AgentQueryRequest(BaseModel):
    """User query submitted to the agent."""
    query: str = Field(..., min_length=1, description="Non-empty user query string")
```

### Response Models (`agent/src/agent/models.py`)

```python
from typing import Literal

class AgentQueryAccepted(BaseModel):
    """Successful acceptance of a query."""
    status: Literal["ok"]
    agent_run_id: str = Field(..., description="Unique run identifier for SSE stream")

class AgentQueryRejected(BaseModel):
    """Rejection due to invalid input."""
    status: Literal["rejected"]
    agent_run_id: str = Field(..., description="Unique run identifier for error stream")
    reason: str = Field(..., description="Reason for rejection (e.g., 'invalid query')")
```

### Run Store Interface (`agent/src/agent/run_store.py`)

```python
from dataclasses import dataclass
from typing import Optional
from asyncio import Queue

@dataclass
class Run:
    """Single agent run."""
    id: str                              # agent_run_id (UUID v4 as hex string)
    query: str
    created_at: float                    # Unix timestamp
    status: str                           # "pending", "completed", "failed"
    events: Queue                        # asyncio.Queue for event messages
    result: Optional[dict] = None        # {"answer": "…"} or {"error": "…"}
    started_at: Optional[float] = None
    ended_at: Optional[float] = None
    in_progress_sent_at: Optional[float] = None

class RunStore:
    """In-memory registry of active runs."""
    
    async def create(query: str) -> Run:
        """Generate unique agent_run_id, create Run, enqueue in_progress event, return Run."""
        # Returns: Run object with id, events queue pre-populated with in_progress
    
    async def get(run_id: str) -> Optional[Run]:
        """Retrieve Run by agent_run_id; return None if not found or expired."""
    
    async def enqueue_event(run_id: str, event_type: str, data: dict) -> None:
        """Append event to run's queue for SSE delivery."""
    
    async def mark_failed(run_id: str, reason: str) -> None:
        """Mark rejected run as failed; enqueue failed event."""
    
    async def cleanup_expired(ttl_seconds: int = 3600) -> None:
        """Delete runs older than ttl_seconds (cron task, deferred to Story 2)."""
```

### Orchestrator Interface (`agent/src/agent/orchestrator.py`)

```python
async def spawn_background_job(agent_run_id: str, query: str) -> None:
    """Spawn background task for this run without waiting for it to complete.
    
    Calls the PER-20 process_question(query) in background.
    Enqueues result events to run_store.
    """
```

### Route Handler (`agent/src/agent/routes.py`)

```python
from fastapi import APIRouter, HTTPException
from agent.models import AgentQueryRequest, AgentQueryAccepted, AgentQueryRejected

router = APIRouter(prefix="/agent", tags=["agent"])

@router.post("/query", response_model=AgentQueryAccepted | AgentQueryRejected)
async def post_agent_query(req: AgentQueryRequest) -> AgentQueryAccepted | AgentQueryRejected:
    """
    Accept a user query immediately and return agent_run_id.
    
    - AC-1: Return HTTP 200 immediately without waiting for process_question.
    - AC-2: Reject if query is missing or empty/whitespace-only; do not spawn background job.
    """
```

---

## Algorithm Notes

### Input Validation

1. Receive `AgentQueryRequest` with `query` field.
2. FastAPI automatic validation (Pydantic):
   - Reject if `query` field is missing → 422 Unprocessable Entity (not our 200 response).
   - Reject if `query` is empty string (`""`).
   - Reject if `query` is `None`.
3. **Custom validation:** Strip whitespace; if result is empty, treat as invalid.
4. **Decision:** HTTP 200 for both acceptance and rejection (per AC-1, AC-2 contract); never 422.

### Acceptance Path (AC-1)

1. Input passes validation.
2. Call `run_store.create(query.strip())`:
   - Generate `agent_run_id` as UUID v4 hex string.
   - Create `Run` object with status `"pending"`.
   - Create `asyncio.Queue` for events.
   - Record `in_progress_sent_at = time.time()`.
   - Enqueue `in_progress` event with data `{"run_id": agent_run_id}`.
   - Return `Run` object.
3. Spawn background job: `asyncio.create_task(spawn_background_job(agent_run_id, query.strip()))`.
   - **Critical:** Do not `await` this call; return immediately.
4. Return HTTP 200 `AgentQueryAccepted(status="ok", agent_run_id=run.id)`.

### Rejection Path (AC-2)

1. Input fails validation (empty or whitespace-only `query`).
2. Call `run_store.create("")`:
   - Generate `agent_run_id` as UUID v4 hex string.
   - Create `Run` object with status `"pending"`.
   - Record `in_progress_sent_at = time.time()`.
   - Enqueue `in_progress` event with data `{"run_id": agent_run_id}`.
   - Return `Run` object.
3. Call `run_store.mark_failed(agent_run_id, "invalid query")`:
   - Update status to `"failed"`.
   - Enqueue `failed` event with data `{"run_id": agent_run_id, "error": "invalid query"}`.
   - This event is queued in `run.events` before POST returns; SSE stream (Story 2) reads and delivers it.
4. **Do NOT spawn background job.**
5. Return HTTP 200 `AgentQueryRejected(status="rejected", agent_run_id=run.id, reason="invalid query")`.

### ID Generation

- Use `uuid.uuid4().hex` → 32-character lowercase hex string (e.g., `"a1b2c3d4e5f6..."`).
- Collision probability: negligible (< 2^-120 for < 1 billion IDs).
- No locking required; UUIDs are globally unique.

### Error Handling

- **Validation error (AC-2):** Caught before background job spawn; `mark_failed()` enqueues `failed` event synchronously.
- **Unhandled exception in POST handler:** Return HTTP 500 (FastAPI default). Story 2 will wrap background job in try/except.

---

## Verify

### Unit Test File: `agent/tests/test_agent_query_accept.py`

#### Test 1: AC-1 — Accept valid query returns ok with agent_run_id

```python
async def test_post_agent_query_valid_returns_ok():
    """Given valid query, POST /agent/query returns ok and agent_run_id immediately."""
    # Setup: create test client, clear run store
    # When: POST /agent/query with {"query": "What is …?"}
    # Then: response status 200, body {"status": "ok", "agent_run_id": "<32-char uuid hex>"}
```

**Verification:**
- Response status is 200.
- Response body contains `status: "ok"`.
- Response body contains `agent_run_id` as non-empty string.
- Response is returned without waiting for `process_question` (time < 100ms).

#### Test 2: AC-1 — Immediate return; does not wait for background job

```python
async def test_post_agent_query_returns_immediately():
    """Given valid query, POST returns before process_question completes."""
    # Setup: mock process_question to sleep 5s
    # When: POST /agent/query with {"query": "…"}
    # Then: response returned in < 100ms (before sleep ends)
    # And: background job continues in background (verify via run_store state)
```

**Verification:**
- Request time to response < 100ms.
- Background task state is `"pending"` immediately after return.
- Background task eventually completes (async verification).

#### Test 3: AC-2 — Empty query returns rejected with reason

```python
async def test_post_agent_query_empty_returns_rejected():
    """Given empty query, POST returns rejected with reason."""
    # When: POST /agent/query with {"query": ""}
    # Then: response status 200, body {"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}
```

**Verification:**
- Response status is 200.
- Response body contains `status: "rejected"`.
- Response body contains `reason: "invalid query"`.
- Response body contains `agent_run_id`.

#### Test 4: AC-2 — Missing query field returns rejected

```python
async def test_post_agent_query_missing_field_returns_rejected():
    """Given missing query field, POST returns rejected."""
    # When: POST /agent/query with {} (empty body)
    # Then: response status 200, body {"status": "rejected", "agent_run_id": "…", "reason": "invalid query"}
```

**Verification:**
- Response status is 200.
- Response body contains `status: "rejected"`.
- Response body contains `reason: "invalid query"`.

#### Test 5: AC-2 — Whitespace-only query returns rejected

```python
async def test_post_agent_query_whitespace_returns_rejected():
    """Given whitespace-only query, POST returns rejected."""
    # When: POST /agent/query with {"query": "   \t\n  "}
    # Then: response status 200, body {"status": "rejected", …}
```

**Verification:**
- Response status is 200.
- Response body contains `status: "rejected"`.

#### Test 6: AC-2 — Rejection does not call LLM or MCP

```python
async def test_post_agent_query_rejection_no_lm_call():
    """Given rejected query, LLM and MCP are not called."""
    # Setup: mock process_question with spy
    # When: POST /agent/query with {"query": ""}
    # Then: process_question is never called
```

**Verification:**
- `process_question` mock call count is 0.
- Background job is not spawned.

#### Test 7: AC-1, AC-2 — Both paths return non-empty agent_run_id

```python
async def test_post_agent_query_both_paths_return_run_id():
    """Given any query (valid or invalid), agent_run_id is returned."""
    # When: POST with valid query
    # Then: agent_run_id is non-empty string
    # When: POST with empty query
    # Then: agent_run_id is non-empty string (different from first)
```

**Verification:**
- Acceptance response has `agent_run_id` with length > 0.
- Rejection response has `agent_run_id` with length > 0.
- Two different calls return different `agent_run_id` values.

#### Test 8: AC-1, AC-2 — agent_run_id is consistent across request and SSE stream

```python
async def test_post_agent_query_run_id_consistent():
    """Given POST response with agent_run_id, SSE stream uses same id."""
    # When: POST /agent/query returns {"agent_run_id": "abc123…"}
    # And: GET /agent/runs/{abc123…}/events is called
    # Then: SSE in_progress event contains {"run_id": "abc123…"} (same id)
```

**Verification:**
- POST response `agent_run_id` matches SSE event `run_id` (delivered by Story 2 SSE handler reading event queue).

---

## Notes

- **No auth required.** Out of scope per AC-6 and requirements.
- **No rate limiting.** Assume upstream FastAPI middleware.
- **UUID v4 collision safety:** Safe for up to billions of runs.
- **Synchronous validation:** O(1) string check; no async overhead; validation happens before task spawn.
- **Background job spawn:** Via `asyncio.create_task()`; does not block request handler.
- **Event enqueue:** Rejection path enqueues `in_progress` and `failed` events to `run.events` (an `asyncio.Queue`) synchronously before POST returns. Story 2 SSE stream handler reads these queued events when `GET /agent/runs/{agent_run_id}/events` is called (see Story 2 for event delivery).
- **PER-20 dependency:** `process_question(query: str) → dict` signature is fixed (AC-6). Any future change must preserve return type.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|----|-----------|---------------|
| AC-1 | 1, 2, 7 | Valid query → immediate ok response | Response contains `status: "ok"`, `agent_run_id`, returned in < 100ms |
| AC-2 | 3, 4, 5, 6, 7 | Empty/missing/whitespace query → immediate rejected response, no LLM/MCP call | Response contains `status: "rejected"`, `reason: "invalid query"`, `agent_run_id`; `process_question` never called; `failed` event enqueued |

**Total Test Count: 8** (matching breakdown and ticket AC count)
```
