```markdown
# Story LLD: 02-agent — Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Ticket ID:** PER-44  
**Story ID:** 02-agent  
**Repository:** `agent`  
**Acceptance Criteria:** AC-3, AC-4, AC-5  
**Story Type:** Backend Feature  

---

## Acceptance Criteria

### AC-3: One event stream per run

**Given** an accepted run  
**When** the caller sends `GET /agent/runs/{agent_run_id}/events` with `Accept: text/event-stream`  
**Then** the first event is `in_progress` with `data: {"run_id": "<agent_run_id>"}`, every subsequent event includes the same `run_id`, and an unknown id returns HTTP 404.

### AC-4: Five-second hold on completed only

**Given** `process_question` returns an answer  
**When** the background job completes  
**Then** if the answer is ready less than 5 seconds after the `in_progress` event, the stream stays on `in_progress` until 5 seconds after that event, then sends `completed` with `data: {"run_id": "<agent_run_id>", "response": {"answer": "<text>"}}` and ends; if the answer is ready at 5 seconds or later, `completed` is sent as soon as the answer is ready and the stream ends. The hold does not delay the POST reply.

### AC-5: Failure is immediate

**Given** `process_question` returns `{"error": "..."}` or the LLM or MCP call fails  
**When** the error is detected  
**Then** the stream sends `failed` with `data: {"run_id": "<agent_run_id>", "error": "<that message>"}` immediately, without waiting for the 5-second hold and without sending `completed`.

---

## File Plan

All paths resolve from the `agent` repository root.

### New Files

```
agent/
├── src/
│   ├── agent/
│   │   ├── sse_handler.py
│   │   └── background_task.py
│   └── main.py (extend with GET route and cleanup task startup)
└── tests/
    └── test_agent_sse_stream.py
```

**Key files:**

1. **`agent/src/agent/sse_handler.py`** — SSE stream reader and event formatter; yields `text/event-stream` format.
2. **`agent/src/agent/background_task.py`** — Background orchestrator; calls `process_question`, enforces 5-second hold, enqueues result events.
3. **`agent/src/main.py`** — Add `GET /agent/runs/{agent_run_id}/events` handler; extend with background cleanup task startup (TTL-based run expiration).
4. **`agent/tests/test_agent_sse_stream.py`** — Unit and integration tests for AC-3, AC-4, AC-5.

### Modified Files

```
agent/
├── src/agent/run_store.py (extend with cleanup_expired and enqueue_event methods)
├── src/agent/models.py (add SSE event data models)
└── src/main.py (add GET route handler and cleanup task startup)
```

---

## Interfaces

### Event Data Models (`agent/src/agent/models.py` — extend)

```python
from typing import Literal
from pydantic import BaseModel

class InProgressEventData(BaseModel):
    """Data for in_progress event."""
    run_id: str

class CompletedEventData(BaseModel):
    """Data for completed event."""
    run_id: str
    response: dict  # {"answer": "<text>"}

class FailedEventData(BaseModel):
    """Data for failed event."""
    run_id: str
    error: str
```

### SSE Handler Interface (`agent/src/agent/sse_handler.py`)

```python
from typing import AsyncGenerator
import json
from fastapi import HTTPException
from agent.run_store import run_store

async def stream_events(run_id: str) -> AsyncGenerator[str, None]:
    """
    Stream events for a given run in SSE format.
    
    Yields lines in SSE format:
      event: <type>
      data: <json>
      
    First event is in_progress.
    Second event is exactly one of (completed | failed).
    Stream closes after second event.
    
    Raises HTTPException(404) if run_id not found.
    """
    run = await run_store.get(run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    async for event_payload in run.events:
        event_type = event_payload["event"]
        data_json = json.dumps(event_payload["data"])
        yield f"event: {event_type}\n"
        yield f"data: {data_json}\n\n"
        
        if event_type in ("completed", "failed"):
            break  # Close stream after final event
```

### Background Task Interface (`agent/src/agent/background_task.py`)

```python
import asyncio
import time
from agent.run_store import run_store
from agent.orchestrator import process_question

async def run_background_job(agent_run_id: str, query: str) -> None:
    """
    Execute process_question in background; enqueue events to run store.
    
    Calls:
      1. run_store.get(agent_run_id)
      2. process_question(query) (from PER-20)
      3. Enforce 5-second hold for completed only
      4. run_store.enqueue_event() for result
    
    On error or LLM/MCP failure:
      - Enqueue failed event immediately (no hold)
      - Do not enqueue completed event
    """
    run = await run_store.get(agent_run_id)
    if not run:
        return  # Run already cleaned up; abort silently
    
    try:
        run.started_at = time.time()
        
        # Call existing PER-20 orchestrator
        result = await process_question(query)
        
        if "error" in result:
            # Failure path (AC-5): send immediately
            await run_store.enqueue_event(agent_run_id, "failed", {
                "run_id": agent_run_id,
                "error": result["error"]
            })
            run.status = "failed"
        else:
            # Success path (AC-4): apply 5-second hold
            elapsed = time.time() - run.in_progress_sent_at
            hold_remaining = 5.0 - elapsed
            
            if hold_remaining > 0:
                await asyncio.sleep(hold_remaining)
            
            await run_store.enqueue_event(agent_run_id, "completed", {
                "run_id": agent_run_id,
                "response": {"answer": result["answer"]}
            })
            run.status = "completed"
        
        run.result = result
    
    except Exception as e:
        # Unhandled exception (AC-5): send failed immediately
        await run_store.enqueue_event(agent_run_id, "failed", {
            "run_id": agent_run_id,
            "error": str(e)
        })
        run.status = "failed"
    
    finally:
        run.ended_at = time.time()
```

### Run Store Interface — Extensions (`agent/src/agent/run_store.py` — extend)

```python
import time

class RunStore:
    """In-memory registry of active runs (from Story 1, extended here)."""
    
    async def enqueue_event(self, run_id: str, event_type: str, data: dict) -> None:
        """Append event to run's event queue for SSE delivery.
        
        Args:
            run_id: agent_run_id
            event_type: "in_progress", "completed", or "failed"
            data: JSON-serializable dict with run_id and result/error
        """
        run = self._runs.get(run_id)
        if run:
            await run.events.put({"event": event_type, "data": data})
    
    async def cleanup_expired(self, ttl_seconds: int = 3600) -> None:
        """Delete runs older than ttl_seconds (Unix timestamp).
        
        Called periodically by cleanup task on app startup.
        Default TTL: 1 hour.
        """
        now = time.time()
        expired_ids = [
            run_id for run_id, run in self._runs.items()
            if (now - run.created_at) > ttl_seconds
        ]
        for run_id in expired_ids:
            del self._runs[run_id]
```

### Route Handler — Extension (`agent/src/main.py` — add to existing app)

```python
from fastapi import FastAPI, HTTPException
from fastapi.responses import StreamingResponse
from agent.sse_handler import stream_events
from agent.run_store import run_store
import asyncio
import os

# Assuming app, run_store initialized in Story 1

@app.get("/agent/runs/{agent_run_id}/events")
async def get_agent_events(agent_run_id: str) -> StreamingResponse:
    """
    Stream SSE events for a given run.
    
    - AC-3: Returns 200 text/event-stream if run exists; 404 if unknown.
    - AC-4, AC-5: Emits in_progress, then (completed | failed), with timing rules enforced.
    """
    run = await run_store.get(agent_run_id)
    if not run:
        raise HTTPException(status_code=404, detail="Run not found")
    
    return StreamingResponse(
        stream_events(agent_run_id),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
        }
    )

# Cleanup task (TTL-based run expiration)
async def cleanup_task():
    """Periodically delete expired runs."""
    ttl_seconds = int(os.getenv("RUN_CLEANUP_TTL_SECONDS", 3600))
    interval_seconds = int(os.getenv("RUN_CLEANUP_INTERVAL_SECONDS", 60))
    while True:
        await asyncio.sleep(interval_seconds)
        await run_store.cleanup_expired(ttl_seconds=ttl_seconds)

@app.on_event("startup")
async def startup():
    """Start cleanup task on app startup."""
    asyncio.create_task(cleanup_task())
```

---

## Algorithm Notes

### SSE Stream Delivery (AC-3)

1. Receive `GET /agent/runs/{agent_run_id}/events`.
2. Call `run_store.get(agent_run_id)`:
   - If `None`: raise `HTTPException(status_code=404, detail="Run not found")`.
   - Else: proceed.
3. Call `stream_events(agent_run_id)` to create async generator.
4. Return `StreamingResponse(stream_events(...), media_type="text/event-stream", headers={...})`.

### Event Generator (`stream_events()`)

1. Loop: `async for event_payload in run.events`:
   - Extract `event_type` (string: "in_progress", "completed", "failed").
   - Extract `data` (dict).
   - Serialize `data` to JSON.
   - Yield SSE line format:
     ```
     event: {event_type}\n
     data: {json_data}\n
     \n
     ```
   - If `event_type` in (`"completed"`, `"failed"`): break (close stream).
2. If an exception occurs or queue is exhausted: close stream (no partial events).

### Background Job Orchestration (AC-4, AC-5)

1. **Spawn context (from Story 1):**
   - `POST /agent/query` calls `asyncio.create_task(run_background_job(agent_run_id, query))`.
   - Returns immediately to caller.

2. **Background task execution:**
   - Retrieves run from store.
   - Calls `process_question(query)` (PER-20 orchestrator).
   - On success: calculates hold time and enqueues `completed` event after hold.
   - On error: enqueues `failed` event immediately (no hold).
   - On exception: catches and enqueues `failed` event with error message.

3. **5-Second Hold Calculation (AC-4):**
   - `in_progress_sent_at` recorded by `run_store.create()` in Story 1.
   - On answer ready: `elapsed = time.time() - in_progress_sent_at`.
   - If `elapsed < 5.0`: `await asyncio.sleep(5.0 - elapsed)`.
   - If `elapsed >= 5.0`: send immediately (no sleep).
   - This enforces: `completed` event sent at time `max(ready_time, in_progress_time + 5.0)`.

4. **Failure Timing (AC-5):**
   - No hold on `failed` events.
   - Sent immediately when error is detected (before 5 seconds elapsed).
   - Same behavior for LLM errors, MCP errors, and validation errors (from Story 1 rejection).

### Cleanup Task (Run Expiration)

1. **Startup (in `main.py`):**
   - Background cleanup task created via `asyncio.create_task(cleanup_task())` on app startup.
   - Runs every interval seconds (default 60).

2. **Cleanup logic (in `run_store.py`):**
   - Iterates `_runs` dict.
   - Deletes runs where `(now - created_at) > ttl_seconds`.
   - Default TTL: 3600 seconds (1 hour).

3. **Effect:**
   - Runs deleted after TTL expiration (default 1 hour after creation).
   - If SSE subscriber requests unknown run: returns HTTP 404.
   - Configurable via `RUN_CLEANUP_TTL_SECONDS` and `RUN_CLEANUP_INTERVAL_SECONDS` env vars.

---

## Tests

### Unit Test File: `agent/tests/test_agent_sse_stream.py`

#### Test 1: AC-3 — GET /agent/runs/{known_id}/events returns HTTP 200 with text/event-stream

```python
async def test_get_agent_events_known_run_returns_200():
    """Given a known run_id, GET /agent/runs/{id}/events returns HTTP 200 with text/event-stream."""
    # Setup: POST /agent/query to create a run; capture agent_run_id
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: response status 200, content-type "text/event-stream"
```

**Verification:**
- Response status is 200.
- Response header `Content-Type` is `text/event-stream`.
- Response header `Cache-Control` is `no-cache`.
- Response header `Connection` is `keep-alive`.

#### Test 2: AC-3 — First event is in_progress with correct run_id

```python
async def test_get_agent_events_first_event_in_progress():
    """Given a known run_id, first SSE event is in_progress with correct run_id."""
    # Setup: POST /agent/query with {"query": "What is …?"}; capture agent_run_id
    # When: GET /agent/runs/{agent_run_id}/events, read first event
    # Then: event type is "in_progress", data is {"run_id": "<agent_run_id>"}
```

**Verification:**
- First event line is `event: in_progress`.
- First data line is `data: {"run_id": "<agent_run_id>"}` (exact match).
- `run_id` in data matches POST response `agent_run_id`.

#### Test 3: AC-3 — All events include same run_id

```python
async def test_get_agent_events_all_events_same_run_id():
    """Given a run, all SSE events include the same run_id."""
    # Setup: POST /agent/query; capture agent_run_id
    # When: GET /agent/runs/{agent_run_id}/events, read all events until stream closes
    # Then: every event data contains "run_id": "<agent_run_id>" (unchanged across events)
```

**Verification:**
- Parse all events from stream.
- Extract `run_id` from each event's data.
- Confirm all `run_id` values match the POST response `agent_run_id`.

#### Test 4: AC-3 — Unknown run_id returns HTTP 404

```python
async def test_get_agent_events_unknown_run_returns_404():
    """Given an unknown run_id, GET /agent/runs/{id}/events returns HTTP 404."""
    # When: GET /agent/runs/{unknown_uuid}/events
    # Then: response status 404, body {"detail": "Run not found"}
```

**Verification:**
- Response status is 404.
- Response body contains `"detail": "Run not found"`.

#### Test 5: AC-3 — Multiple concurrent subscribers receive identical events

```python
async def test_get_agent_events_multiple_subscribers_same_order():
    """Given a run, multiple concurrent SSE subscribers receive identical events in same order."""
    # Setup: POST /agent/query; capture agent_run_id; mock process_question to delay 2s
    # When: open two concurrent SSE streams for same agent_run_id
    # Then: both streams receive in_progress, completed in same order; events are identical
```

**Verification:**
- Launch two concurrent `asyncio.Task` for same `agent_run_id`.
- Collect events from both streams.
- Confirm event order and content are identical.

#### Test 6: AC-4 — Fast answer (< 5s): hold until 5s, send completed at t ≥ 5s

```python
async def test_get_agent_events_fast_answer_5s_hold():
    """Given an answer ready < 5s after in_progress, stream held until 5s, then completed sent."""
    # Setup: POST /agent/query; mock process_question to return answer in 0.1s
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: in_progress sent at t=0, completed sent at t ≥ 5.0 (held for ~4.9s)
```

**Verification:**
- Record timestamp of `in_progress` event receipt (`t_in_progress`).
- Record timestamp of `completed` event receipt (`t_completed`).
- Assert `(t_completed - t_in_progress) >= 4.9` (allow 100ms jitter).
- Assert `completed` event data contains answer.

#### Test 7: AC-4 — Slow answer (≥ 5s): send completed immediately when ready (no extra hold)

```python
async def test_get_agent_events_slow_answer_no_extra_hold():
    """Given an answer ready ≥ 5s after in_progress, completed sent immediately when ready."""
    # Setup: POST /agent/query; mock process_question to return answer in 6s
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: in_progress sent at t=0, completed sent at t ≈ 6.0 (no extra hold)
```

**Verification:**
- Record timestamp of `in_progress` event receipt (`t_in_progress`).
- Record timestamp of `completed` event receipt (`t_completed`).
- Assert `(t_completed - t_in_progress) >= 5.9` and `<= 6.1` (tight range, no extra hold).

#### Test 8: AC-4 — Edge case: answer ready exactly at 5s

```python
async def test_get_agent_events_answer_at_5s_exact():
    """Given an answer ready exactly at 5s after in_progress, completed sent immediately."""
    # Setup: POST /agent/query; mock process_question to return answer in exactly 5.0s
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: in_progress at t=0, completed at t ≈ 5.0 (no extra hold)
```

**Verification:**
- Record timestamps.
- Assert `(t_completed - t_in_progress) >= 4.95` and `<= 5.05` (tight tolerance at boundary).

#### Test 9: AC-5 — LLM error: in_progress, then failed immediately (no hold, no completed)

```python
async def test_get_agent_events_llm_error_immediate_failed():
    """Given LLM error, stream sends in_progress, then failed immediately (no 5s hold)."""
    # Setup: POST /agent/query; mock process_question to return {"error": "LLM timeout"}
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: in_progress sent at t=0, failed sent at t < 1.0 (immediately, no 5s hold)
    # And: no completed event sent
```

**Verification:**
- Parse events: first is `in_progress`, second is `failed`.
- No `completed` event present.
- Time from `in_progress` to `failed` is < 1.0 second.
- `failed` event data contains `"error": "LLM timeout"`.

#### Test 10: AC-5 — MCP error: in_progress, then failed immediately (no hold, no completed)

```python
async def test_get_agent_events_mcp_error_immediate_failed():
    """Given MCP error, stream sends in_progress, then failed immediately."""
    # Setup: POST /agent/query; mock process_question to raise exception (MCP call fails)
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: in_progress sent at t=0, failed sent at t < 1.0, no completed
```

**Verification:**
- Parse events: `in_progress`, then `failed`.
- No `completed` event.
- Elapsed time < 1.0 second.
- `failed` event data contains error message (exception string).

#### Test 11: AC-5 — Invalid query rejection: SSE stream sends failed immediately (no completed)

```python
async def test_get_agent_events_invalid_query_failed_no_wait():
    """Given rejected query from POST, SSE sends in_progress then failed immediately."""
    # Setup: POST /agent/query with {"query": ""}; capture agent_run_id
    # When: GET /agent/runs/{agent_run_id}/events
    # Then: first event is in_progress, second event is failed (from AC-2 rejection path)
    # And: no completed event sent, error is "invalid query"
```

**Verification:**
- First event is `in_progress`.
- Second event is `failed`.
- No `completed` event.
- `failed` event data contains `"error": "invalid query"`.

#### Test 12: AC-4 — Stream closes after completed event

```python
async def test_get_agent_events_stream_closes_after_completed():
    """Given completed event sent, stream closes and does not send further events."""
    # Setup: POST /agent/query with valid query
    # When: GET /agent/runs/{agent_run_id}/events, read until stream closes
    # Then: last event is completed, stream is closed, no additional events
```

**Verification:**
- Read events until stream closes (no more data).
- Last event received is `completed`.
- Count of events is exactly 2 (in_progress, completed).

#### Test 13: AC-5 — Stream closes after failed event

```python
async def test_get_agent_events_stream_closes_after_failed():
    """Given failed event sent, stream closes and does not send further events."""
    # Setup: POST /agent/query; mock process_question to return error
    # When: GET /agent/runs/{agent_run_id}/events, read until stream closes
    # Then: last event is failed, stream is closed, no additional events
```

**Verification:**
- Read events until stream closes.
- Last event received is `failed`.
- Count of events is exactly 2 (in_progress, failed).

#### Test 14: AC-3 — Subscriber disconnect does not cancel background job

```python
async def test_get_agent_events_disconnect_does_not_cancel_job():
    """Given a subscriber disconnects mid-stream, background job continues and completes."""
    # Setup: POST /agent/query; mock process_question to delay 2s; open SSE stream
    # When: close SSE stream after receiving in_progress (disconnect subscriber)
    # Then: background job continues and completes in background
    # And: a new subscriber opening same run_id can retrieve completed event
```

**Verification:**
- Open SSE stream, read first event, close connection.
- Wait 3 seconds (allow background job to complete).
- Open new SSE stream for same run_id.
- New subscriber can retrieve in_progress and completed events (replay from queue).

#### Test 15: AC-3 — Late subscriber retrieves all events (no replay loss)

```python
async def test_get_agent_events_late_subscriber_gets_all_events():
    """Given background job completes before subscriber connects, new subscriber gets all events."""
    # Setup: POST /agent/query; mock process_question to return answer in 0.5s
    # When: wait 1 second (allow job to complete), then open SSE stream
    # Then: subscriber receives in_progress, then completed (no events lost)
```

**Verification:**
- Verify events are queued in `run.events` before any subscriber connects.
- New subscriber drains queue and receives full event history.
- No events are dropped.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|--------|-----------|---------------|
| AC-3 | 1, 2, 3, 4, 5, 14, 15 | Stream setup, event delivery, run lookup, multi-subscriber | HTTP 200, `text/event-stream`, first event `in_progress`, all events same `run_id`, 404 on unknown, subscribers independent, events not lost |
| AC-4 | 6, 7, 8, 12 | 5-second hold for `completed` only | Fast answer held until 5s, slow answer immediate, edge case at 5s, stream closes after `completed` |
| AC-5 | 9, 10, 11, 13 | `failed` immediate, no hold, no `completed` | LLM error, MCP error, invalid query, all send `failed` immediately; no `completed` event sent; stream closes after `failed` |

**Total Test Count: 15**

---

## Notes

- **Event Queue (Run.events):** `asyncio.Queue` per run; events enqueued synchronously by background task before subscribers open; replay available to late subscribers.
- **5-Second Hold:** Enforced in background task via `asyncio.sleep()`; does not block event stream reader or other jobs.
- **Timestamp Recording:** `in_progress_sent_at` recorded by Story 1 `run_store.create()`; used by background task for hold calculation.
- **Cleanup Policy:** Default 1-hour TTL; configurable via `RUN_CLEANUP_TTL_SECONDS` and `RUN_CLEANUP_INTERVAL_SECONDS` env vars.
- **Event Format:** SSE spec: `event: <type>\ndata: <json>\n\n` (two newlines after data).
- **Exception Handling:** Any unhandled exception in background task caught and converted to `failed` event with error message as string.
- **PER-20 Dependency:** `process_question(query: str) → dict` signature is stable (AC-6). Expected return: `{"answer": "<text>"}` or `{"error": "<message>"}`.
- **Process Isolation:** MCP calls remain in main agent process; no child processes forked by background task or SSE stream.
- **Backward Compatibility:** Story 2 does not modify POST /query or GET /health; see Story 3 (03-agent) for audit.
- **Routes Location:** Per `change_requests.md`, all routes (POST /agent/query, GET /agent/runs/{id}/events, existing POST /query and GET /health) are registered in `agent/src/main.py`. No separate routes module.
```
