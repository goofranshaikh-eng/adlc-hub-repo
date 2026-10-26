```markdown
# Breakdown: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Overview

This breakdown translates the epic's [requirements.md](./requirements.md) and [hld.md](./hld.md) into three ordered, implementable stories. Each story covers one or more acceptance criteria from the ticket, with concrete file paths, allowed implementation patterns, and verification steps that build toward the epic's definition of done.

**Epic Acceptance Criteria:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6  
**Story Count:** 3  
**Total Test Count:** 32 (8 + 15 + 9, see Verification Approach per story)

---

## Story 1: Accept FastAPI query without blocking; return agent_run_id immediately

**Repo:** `agent`

**Acceptance Criteria:**
- AC-1: Immediate accept
- AC-2: Reject without a run

**Description:**

Implement the `POST /agent/query` endpoint to accept a user query immediately and return an HTTP 200 response with `status` and `agent_run_id` before any LLM or MCP call. The handler validates the input synchronously: if `query` is missing, `None`, or empty/whitespace-only, reject with status `"rejected"` and reason `"invalid query"`, enqueue a `failed` event to the run store (see Story 2 for SSE delivery), and do not spawn a background job. On acceptance, generate a unique `agent_run_id`, enqueue an `in_progress` event, and spawn the background orchestrator without waiting for it to complete. See [specs/1.md](./specs/1.md) for detailed schema, API contracts, and implementation notes.

**Allowed Paths:**
- `agent/src/agent/__init__.py` — module marker
- `agent/src/agent/models.py` — Pydantic request/response models (`AgentQueryRequest`, `AgentQueryAccepted`, `AgentQueryRejected`)
- `agent/src/agent/run_store.py` — in-memory run registry with `create()`, `get()`, `enqueue_event()`, `mark_failed()` methods
- `agent/src/agent/orchestrator.py` — background job spawn logic
- `agent/src/agent/routes.py` — POST /agent/query route handler
- `agent/src/main.py` — register agent router; verify no interference with existing POST /query and GET /health
- `agent/tests/test_agent_query_accept.py` — unit tests for AC-1 and AC-2

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | POST /agent/query with valid query returns HTTP 200 `{"status": "ok", "agent_run_id": "<uuid>"}` | AC-1 | 1 |
| 2 | POST /agent/query accepts immediately; does not wait for process_question | AC-1 | 1 |
| 3 | POST /agent/query with empty query returns HTTP 200 `{"status": "rejected", "agent_run_id": "<uuid>", "reason": "invalid query"}` | AC-2 | 1 |
| 4 | POST /agent/query with missing query field returns rejected response | AC-2 | 1 |
| 5 | POST /agent/query with whitespace-only query returns rejected response | AC-2 | 1 |
| 6 | Rejection path does not call LLM or MCP server | AC-2 | 1 |
| 7 | Both acceptance and rejection return a non-empty agent_run_id | AC-1, AC-2 | 1 |
| 8 | agent_run_id is consistent (same id used in subsequent SSE stream) | AC-1, AC-2 | 1 |

**Total Test Count: 8**

**Acceptance Criteria Verification:**
- AC-1: Tests 1, 2, 7 verify immediate accept and agent_run_id return.
- AC-2: Tests 3, 4, 5, 6, 7 verify rejection, reason field, and no LLM/MCP calls. Rejection path enqueues `failed` event (Story 2 delivers via SSE).

**Notes:**
- `agent_run_id` generated as UUID v4 (hex, 32 chars) for collision-safety.
- Input validation is synchronous (O(1) string check); no async overhead.
- On rejection: call `run_store.mark_failed(agent_run_id, "invalid query")` to enqueue `failed` event before returning.
- Background job spawned via `asyncio.create_task()` (see Story 2 for execution details).
- Run store initialization happens on app startup; cleanup via TTL (deferred to Story 2).
- No auth required (out of scope per requirements).

---

## Story 2: Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Repo:** `agent`

**Acceptance Criteria:**
- AC-3: One event stream
- AC-4: Five-second hold on completed only
- AC-5: Failure is immediate

**Description:**

Implement the `GET /agent/runs/{agent_run_id}/events` endpoint to serve Server-Sent Events for a given run. The handler validates the run exists (404 if unknown), then streams events in order: `in_progress` is sent immediately upon run creation (enqueued by Story 1), followed by exactly one of `completed` or `failed`. The background orchestrator calls the existing `process_question(query)` function and enqueues the result. If the result is an answer, apply a 5-second hold only to the `completed` event: if the answer arrives < 5 seconds after `in_progress`, wait until 5 seconds have elapsed; if it arrives >= 5 seconds, send immediately. If the result is an error or an exception occurs, enqueue `failed` immediately (no hold) and close the stream. Multiple SSE subscribers are allowed for the same run_id; they all receive the same events in the same order. See [specs/2.md](./specs/2.md) for detailed schema, API contracts, and implementation notes.

**Allowed Paths:**
- `agent/src/agent/run_store.py` — extend with `enqueue_event()`, `cleanup_expired()`, event queue per run
- `agent/src/agent/sse_handler.py` — SSE stream generator; formats events as `event: <type>\ndata: <json>\n\n`
- `agent/src/agent/background_task.py` — background orchestrator; calls `process_question(query)`, enforces 5-second hold, enqueues result events
- `agent/src/agent/routes.py` — extend with GET /agent/runs/{agent_run_id}/events handler; returns `StreamingResponse`
- `agent/src/main.py` — register cleanup task; start TTL-based run cleanup on app startup
- `agent/tests/test_agent_sse_stream.py` — unit and integration tests for AC-3, AC-4, AC-5

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | GET /agent/runs/{known_id}/events returns HTTP 200 with `text/event-stream` media type | AC-3 | 1 |
| 2 | First event is `in_progress` with `data: {"run_id": "<agent_run_id>"}` | AC-3 | 1 |
| 3 | All subsequent events include the same `run_id` | AC-3 | 1 |
| 4 | GET /agent/runs/{unknown_id}/events returns HTTP 404 `{"detail": "Run not found"}` | AC-3 | 1 |
| 5 | Multiple concurrent SSE subscribers for same run_id receive identical events in order | AC-3 | 1 |
| 6 | Fast answer (ready < 5s after in_progress): stream held on in_progress, completed sent at t >= 5s | AC-4 | 1 |
| 7 | Slow answer (ready >= 5s after in_progress): in_progress at t=0, completed sent at t >= answer_ready_time (no extra hold) | AC-4 | 1 |
| 8 | Edge case: answer ready exactly at 5s receives completed immediately (no extra hold) | AC-4 | 1 |
| 9 | LLM error: stream sends in_progress, then failed (no completed), elapsed < 5s | AC-5 | 1 |
| 10 | MCP error: stream sends in_progress, then failed (no completed), elapsed < 5s | AC-5 | 1 |
| 11 | Invalid query rejection (from Story 1): GET /agent/runs/{id}/events sends failed immediately, no completed | AC-5 | 1 |
| 12 | Stream closes after completed event | AC-4 | 1 |
| 13 | Stream closes after failed event | AC-5 | 1 |
| 14 | Background job is not cancelled when a subscriber disconnects mid-stream | AC-3 | 1 |
| 15 | Second subscriber attaching after job completes can still open SSE and retrieve all events | AC-3 | 1 |

**Total Test Count: 15**

**Acceptance Criteria Verification:**
- AC-3: Tests 1, 2, 3, 4, 5, 14, 15 verify event stream delivery, correct run_id, 404 on unknown, and subscription independence.
- AC-4: Tests 6, 7, 8, 12 verify 5-second hold logic and stream closure.
- AC-5: Tests 9, 10, 11, 13 verify immediate error delivery and stream closure.

**Notes:**
- Event queue: in-memory `asyncio.Queue` per run (PoC; Redis for multi-process).
- 5-second hold enforced in background task via `asyncio.sleep()`, not in stream reader (non-blocking).
- `in_progress_sent_at` timestamp recorded in `Run` object to calculate hold duration.
- TTL-based cleanup (default 1 hour): delete runs after completion/failure to prevent memory leak.
- Cleanup task runs every 60 seconds; configurable via `RUN_CLEANUP_INTERVAL_SECONDS` env var.
- Stream format: `event: <type>\ndata: <json>\n\n` per SSE spec.
- Exception handling: any unhandled exception in background task enqueues `failed` event with exception message.

---

## Story 3: Preserve existing POST /query and GET /health endpoints

**Repo:** `agent`

**Acceptance Criteria:**
- AC-6: Existing /query path unchanged

**Description:**

Audit and verify that the existing `POST /query` and `GET /health` endpoints remain unchanged after Stories 1 and 2 are implemented. The `POST /query` handler must continue to block and call `process_question(question)` directly, returning `{"answer": "..."}` or `{"error": "..."}` synchronously. `GET /health` must return `{"status": "ok"}` unchanged. Verify that MCP is still called only from the main agent process (no child processes or threads), and that tool names, id normalization, and `MCP_URL` are unchanged. Confirm coexistence: both POST /agent/query (new, non-blocking) and POST /query (existing, blocking) work correctly in the same session without interference. See [specs/3.md](./specs/3.md) for detailed verification scope and implementation notes.

**Allowed Paths:**
- `agent/src/main.py` (or `agent/src/routes/query.py` if routes are modularized) — existing POST /query and GET /health handlers; no changes
- `agent/tests/test_backward_compat.py` — verification tests for AC-6

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | POST /query with valid question returns HTTP 200 `{"answer": "<text>"}` blocking | AC-6 | 1 |
| 2 | POST /query with error condition returns HTTP 200 `{"error": "<message>"}` blocking | AC-6 | 1 |
| 3 | POST /query latency is unchanged (no regression from Stories 1–2 overhead) | AC-6 | 1 |
| 4 | GET /health returns HTTP 200 `{"status": "ok"}` | AC-6 | 1 |
| 5 | POST /query and POST /agent/query coexist; both respond independently | AC-6 | 1 |
| 6 | MCP HTTP calls originate only from main agent process (no child processes spawned) | AC-6 | 1 |
| 7 | Tool names used in POST /query remain unchanged | AC-6 | 1 |
| 8 | ID normalization logic (if any) in POST /query is unchanged | AC-6 | 1 |
| 9 | MCP_URL configuration is unchanged | AC-6 | 1 |

**Total Test Count: 9**

**Acceptance Criteria Verification:**
- AC-6: All 9 tests verify existing POST /query and GET /health behavior, coexistence with new endpoints, and PER-20 contract stability.

**Notes:**
- No new code in existing handlers; only verification and tests.
- Baseline latency measured before Stories 1–2 merge; regression test added to prevent performance degradation.
- Coexistence test exercises both endpoints in same session to confirm no global state corruption.
- MCP process isolation verified: `process_question` called only from main agent process (no threading or multiprocessing).
- Out of scope: auth (same as PER-20), rate limiting, metrics/logging.

---

## Execution Order & Dependencies

```
Story 1 (Accept & validate)
  ↓ (must complete before Story 2 starts)
Story 2 (Stream events & background orchestration)
  ↓ (must complete before Story 3 starts)
Story 3 (Backward compatibility verification)
```

**Rationale:**
- Story 1 establishes the POST /agent/query handler and run store; Story 2 depends on `run_store.enqueue_event()` and `agent_run_id`.
- Story 2 implements the background orchestrator and SSE stream; Story 3 verifies it does not interfere with existing endpoints.
- All three must be complete before the epic is marked done.

---

## Coverage & Verification

### Acceptance Criteria Mapping

| AC | Stories | Test Count |
|----|---------|-----------|
| AC-1: Immediate accept | Story 1 | 2 |
| AC-2: Reject without a run | Story 1 | 6 |
| AC-3: One event stream | Story 2 | 7 |
| AC-4: Five-second hold | Story 2 | 4 |
| AC-5: Failure is immediate | Story 2 | 3 |
| AC-6: Existing paths unchanged | Story 3 | 9 |
| **Total** | **3 stories** | **32 tests** |

**Note:** The breakdown specifies 32 total tests (8 + 15 + 9); the epic's definition of done requires tests covering all six ACs and the cases listed: immediate return, rejection, fast and slow completed, tool/LLM failure, unknown run id, and existing POST /query answer. All are covered.

---

## Epic Definition of Done Alignment

From the ticket, definition of done:

> `POST /agent/query` returns `ok` or `rejected` without waiting for the model. Accepted runs execute the [PER-20](https://linear.app/personalgs/issue/PER-20/integrate-agent-with-mcp-server) orchestrator in the background. The SSE stream emits `in_progress`, then exactly one of `completed` or `failed`, with the 5-second rule applied only to `completed`. Tests cover immediate return, rejection, the fast and slow completed cases, a tool or LLM failure, an unknown run id, and the existing `POST /query` answer.

**Coverage:**
- ✓ Immediate return & rejection: Story 1 (Tests 1–8)
- ✓ Background orchestration: Story 2 (Tests 6–11, via `background_task.py`)
- ✓ SSE stream with `in_progress`, `completed`, `failed`: Story 2 (Tests 1–13)
- ✓ 5-second hold rule: Story 2 (Tests 6–8, 12)
- ✓ Fast and slow completed cases: Story 2 (Tests 6–8)
- ✓ Tool/LLM failure: Story 2 (Tests 9–10)
- ✓ Unknown run id: Story 2 (Test 4), Story 3 (implicit)
- ✓ Existing POST /query: Story 3 (Tests 1–3, 5, 9)

---

## Notes & Open Questions

**Assumptions:**
- `process_question(query: str) → dict` signature and return type are stable per PER-20.
- UUID v4 collision probability is negligible; no locking required for id generation.
- In-memory run store is acceptable for PoC; multi-process deployments will upgrade to Redis.
- 1-hour TTL for run cleanup is reasonable; configurable via env var if needed.

**Deferred:**
- Run history API (`GET /agent/runs`) — not in scope.
- Auth/authorization — out of scope per requirements.
- Metrics/logging beyond startup — out of scope.
- Rate limiting — assume upstream FastAPI middleware.
```
