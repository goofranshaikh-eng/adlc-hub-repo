```markdown
# Breakdown: Accept a FastAPI query immediately and publish the PER-20 answer on SSE

## Overview

This breakdown translates the epic's [requirements.md](./requirements.md) and [hld.md](./hld.md) into three ordered, implementable stories. Each story covers one or more acceptance criteria from the ticket, with concrete file paths, allowed implementation patterns, and verification steps that build toward the epic's definition of done.

**Epic Acceptance Criteria:** AC-1, AC-2, AC-3, AC-4, AC-5, AC-6  
**Story Count:** 3  
**Total Test Count:** 32 (8 + 15 + 9, see Verification Approach per story)

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
- Story 1 establishes the `POST /agent/query` handler and run store with synchronous event enqueue (`in_progress` and rejection `failed` events); Story 2 depends on existing `run_store.create()`, `run_store.mark_failed()`, and `run_store.enqueue_event()` to deliver events via SSE.
- Story 2 implements the background orchestrator, SSE stream handler, and TTL cleanup; Story 3 verifies it does not interfere with existing endpoints.
- All three must be complete before the epic is marked done.

---

## Story 1: Accept FastAPI query without blocking; return agent_run_id immediately

**Repo:** `agent`

**Acceptance Criteria:**
- AC-1: Immediate accept
- AC-2: Reject without a run

**Description:**

Implement the `POST /agent/query` endpoint in `agent/src/main.py` to accept a user query immediately and return an HTTP 200 response with `status` and `agent_run_id` before any LLM or MCP call. The handler validates the input synchronously: if `query` is missing, `None`, or empty/whitespace-only, reject with status `"rejected"` and reason `"invalid query"`, create a run, enqueue both `in_progress` and `failed` events synchronously, and do not spawn a background job. On acceptance, generate a unique `agent_run_id`, create a run, enqueue an `in_progress` event synchronously, and spawn the background orchestrator without waiting for it to complete. See `stories/01-agent.md` for detailed schema, API contracts, and implementation notes.

**Allowed Paths:**
- `agent/src/agent/__init__.py` — module marker
- `agent/src/agent/models.py` — Pydantic request/response models (`AgentQueryRequest`, `AgentQueryAccepted`, `AgentQueryRejected`)
- `agent/src/agent/run_store.py` — in-memory run registry with `create()`, `get()`, `enqueue_event()`, `mark_failed()` methods
- `agent/src/agent/background_task.py` — background job spawn logic
- `agent/src/main.py` — implement `POST /agent/query` handler; verify no interference with existing `POST /query` and `GET /health`
- `agent/tests/test_agent_query_accept.py` — unit tests for AC-1 and AC-2

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | POST /agent/query with valid query returns HTTP 200 `{"status": "ok", "agent_run_id": "<uuid>"}` immediately | AC-1 | 1 |
| 2 | POST /agent/query accepts and returns before `process_question` is invoked (response < 100ms) | AC-1 | 1 |
| 3 | POST /agent/query with empty query returns HTTP 200 `{"status": "rejected", "agent_run_id": "<uuid>", "reason": "invalid query"}` | AC-2 | 1 |
| 4 | POST /agent/query with missing query field returns HTTP 200 `{"status": "rejected", …, "reason": "invalid query"}` | AC-2 | 1 |
| 5 | POST /agent/query with whitespace-only query returns HTTP 200 `{"status": "rejected", …}` | AC-2 | 1 |
| 6 | Rejection path does not spawn background job and does not call `process_question` | AC-2 | 1 |
| 7 | Both acceptance and rejection paths return a non-empty `agent_run_id` and `agent_run_id` differs across calls | AC-1, AC-2 | 1 |
| 8 | Run store contains created run with `id` matching POST response `agent_run_id`; rejection path enqueues `in_progress` and `failed` events synchronously before POST returns | AC-1, AC-2 | 1 |

**Total Test Count: 8**

**Acceptance Criteria Verification:**
- AC-1: Tests 1, 2, 7 verify immediate accept, no wait for `process_question`, and `agent_run_id` return.
- AC-2: Tests 3, 4, 5, 6, 7, 8 verify rejection, reason field, no background job spawn, and synchronous event enqueue. Rejection path enqueues `failed` event for Story 2 SSE delivery.

**Notes:**
- `agent_run_id` generated as UUID v4 (hex, 32 chars) for collision-safety.
- Input validation is synchronous (O(1) string check: strip whitespace, test empty); no async overhead.
- On rejection: call `run_store.create("")` then `run_store.mark_failed(agent_run_id, "invalid query")` to enqueue both `in_progress` and `failed` events synchronously before returning HTTP 200.
- On acceptance: call `run_store.create(query)` to enqueue `in_progress` event, then `asyncio.create_task(run_background_job(agent_run_id, query))` without awaiting.
- Background job implementation deferred to Story 2; spawn call occurs in Story 1.
- Run store initialization happens on app startup; cleanup via TTL deferred to Story 2.
- No auth required (out of scope per requirements).

---

## Story 2: Stream PER-20 answer via SSE with in_progress, completed, or failed events

**Repo:** `agent`

**Acceptance Criteria:**
- AC-3: One event stream
- AC-4: Five-second hold on completed only
- AC-5: Failure is immediate

**Description:**

Implement the `GET /agent/runs/{agent_run_id}/events` endpoint in `agent/src/main.py` to serve Server-Sent Events for a given run. The handler validates the run exists (404 if unknown), then streams events in order: `in_progress` was enqueued by Story 1, followed by exactly one of `completed` or `failed`. Implement `agent/src/agent/sse_handler.py` to read from the run's event queue and yield SSE-formatted events. Implement `agent/src/agent/background_task.py` to call the existing `process_question(query)` function and enqueue the result. If the result is an answer, apply a 5-second hold only to the `completed` event: if the answer is ready < 5 seconds after `in_progress` was enqueued, wait until 5 seconds have elapsed; if it is ready >= 5 seconds after, send immediately. If the result is an error or an exception occurs, enqueue `failed` immediately (no hold) and close the stream. Multiple SSE subscribers are allowed for the same `run_id`; they all receive the same events in the same order. Add a cleanup task to delete expired runs (TTL-based, default 1 hour). See `stories/02-agent.md` for detailed schema, API contracts, and implementation notes.

**Allowed Paths:**
- `agent/src/agent/run_store.py` — extend `enqueue_event()`, add `cleanup_expired()` method
- `agent/src/agent/sse_handler.py` — SSE stream generator; formats events as `event: <type>\ndata: <json>\n\n`
- `agent/src/agent/background_task.py` — implement `run_background_job(agent_run_id, query)`; calls `process_question(query)`, enforces 5-second hold, enqueues result events, handles exceptions
- `agent/src/main.py` — add `GET /agent/runs/{agent_run_id}/events` handler returning `StreamingResponse`; add cleanup task startup on app startup
- `agent/tests/test_agent_sse_stream.py` — unit and integration tests for AC-3, AC-4, AC-5

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | GET /agent/runs/{known_id}/events returns HTTP 200 with `Content-Type: text/event-stream` and appropriate cache/connection headers | AC-3 | 1 |
| 2 | First SSE event is `in_progress` with `data: {"run_id": "<agent_run_id>"}` (enqueued by Story 1) | AC-3 | 1 |
| 3 | All subsequent events include the same `run_id` in their data | AC-3 | 1 |
| 4 | GET /agent/runs/{unknown_id}/events returns HTTP 404 `{"detail": "Run not found"}` | AC-3 | 1 |
| 5 | Multiple concurrent SSE subscribers for same `run_id` receive identical events in same order (no desync) | AC-3 | 1 |
| 6 | Fast answer (ready < 5s after `in_progress` enqueue): `completed` held and sent at t >= 5s from `in_progress` | AC-4 | 1 |
| 7 | Slow answer (ready >= 5s after `in_progress` enqueue): `completed` sent immediately when ready (no extra hold) | AC-4 | 1 |
| 8 | Edge case: answer ready exactly at 5s receives `completed` immediately (no extra hold) | AC-4 | 1 |
| 9 | LLM error: stream sends `in_progress`, then `failed` (no `completed`), elapsed < 1s | AC-5 | 1 |
| 10 | MCP error: stream sends `in_progress`, then `failed` (no `completed`), elapsed < 1s | AC-5 | 1 |
| 11 | Invalid query rejection (from Story 1): GET /agent/runs/{id}/events sends `in_progress` then `failed` (no `completed`) with error: "invalid query" | AC-5 | 1 |
| 12 | Stream closes after `completed` event (exactly 2 events: `in_progress`, `completed`) | AC-4 | 1 |
| 13 | Stream closes after `failed` event (exactly 2 events: `in_progress`, `failed`) | AC-5 | 1 |
| 14 | Background job continues when subscriber disconnects mid-stream (no cancellation) | AC-3 | 1 |
| 15 | New subscriber opening SSE after job completes retrieves all queued events (in_progress and result) with no loss | AC-3 | 1 |

**Total Test Count: 15**

**Acceptance Criteria Verification:**
- AC-3: Tests 1, 2, 3, 4, 5, 14, 15 verify event stream delivery, correct `run_id` in all events, 404 on unknown run, multi-subscriber independence, and event replay.
- AC-4: Tests 6, 7, 8, 12 verify 5-second hold logic applied only to `completed`, timing edge cases, and stream closure.
- AC-5: Tests 9, 10, 11, 13 verify immediate error delivery (no 5-second hold), no `completed` event sent on error, and stream closure.

**Notes:**
- Event queue: in-memory `asyncio.Queue` per run (proof-of-concept; upgrade to Redis for multi-process deployment).
- 5-second hold enforced in background task via `asyncio.sleep()`; non-blocking per-run, does not block other runs or event queue.
- `in_progress_sent_at` timestamp recorded by `run_store.create()` in Story 1; used by background task for hold duration calculation.
- TTL-based cleanup: default 1 hour; runs deleted after `(now - created_at) > ttl_seconds`. Configurable via `RUN_CLEANUP_TTL_SECONDS` env var.
- Cleanup task runs every 60 seconds; configurable via `RUN_CLEANUP_INTERVAL_SECONDS` env var.
- Stream format: SSE spec compliance: `event: <type>\ndata: <json>\n\n` (two newlines after data).
- Exception handling: any unhandled exception in `run_background_job()` caught, `failed` event enqueued with exception message as error string.

---

## Story 3: Preserve existing POST /query and GET /health endpoints

**Repo:** `agent`

**Acceptance Criteria:**
- AC-6: Existing /query path unchanged

**Description:**

Audit and verify that the existing `POST /query` and `GET /health` endpoints in `agent/src/main.py` remain unchanged after Stories 1 and 2 are implemented. The `POST /query` handler must continue to block and call `process_question(question)` directly, returning `{"answer": "..."}` or `{"error": "..."}` synchronously. `GET /health` must return `{"status": "ok"}` unchanged. Verify that MCP is still called only from the main agent process (no child processes or threads), and that tool names, id normalization, and `MCP_URL` are unchanged. Confirm coexistence: both `POST /agent/query` (new, non-blocking, from Stories 1–2) and `POST /query` (existing, blocking) work correctly in the same session without interference. See `stories/03-agent.md` for detailed verification scope and implementation notes.

**Allowed Paths:**
- `agent/src/main.py` — existing `POST /query` and `GET /health` handlers; audit only, no code changes
- `agent/tests/test_backward_compat.py` — verification tests for AC-6

**Verification Approach:**

| # | Test | AC | Count |
|---|------|----|----|
| 1 | POST /query with valid question returns HTTP 200 `{"answer": "<text>"}` with blocking behavior | AC-6 | 1 |
| 2 | POST /query with error condition returns HTTP 200 `{"error": "<message>"}` with blocking behavior | AC-6 | 1 |
| 3 | POST /query latency unchanged (mean post-implementation <= mean baseline * 1.1, 10% tolerance) | AC-6 | 1 |
| 4 | GET /health returns HTTP 200 `{"status": "ok"}` with no side effects | AC-6 | 1 |
| 5 | POST /agent/query (non-blocking, from Stories 1–2) and POST /query (blocking) coexist and respond independently in same session | AC-6 | 1 |
| 6 | MCP HTTP calls originate only from main agent process; no child processes spawned | AC-6 | 1 |
| 7 | Tool names used in POST /query remain unchanged (exact match to baseline) | AC-6 | 1 |
| 8 | ID normalization logic (if any) in POST /query behavior unchanged (exact match to baseline) | AC-6 | 1 |
| 9 | MCP_URL configuration unchanged (exact match to baseline value) | AC-6 | 1 |

**Total Test Count: 9**

**Acceptance Criteria Verification:**
- AC-6: All 9 tests verify existing `POST /query` and `GET /health` behavior, coexistence with new non-blocking endpoints, and PER-20 contract stability (tool names, MCP URL, process isolation).

**Notes:**
- No new code in existing handlers; audit and regression testing only.
- Baseline latency, tool names, ID normalization, and MCP_URL must be captured before Stories 1–2 implementation and documented in test fixtures (e.g., `conftest.py` or test module constants).
- Coexistence test (Test 5) exercises both blocking and non-blocking endpoints in same session to confirm no global state corruption.
- MCP process isolation (Test 6) verified by instrumenting `process_question` or MCP HTTP client to capture process ID; confirm all calls from main FastAPI process PID, no child processes.
- Latency regression test (Test 3) allows 10% tolerance for normal variance (JIT, system load); only fails if Stories 1–2 introduce significant overhead.
- Out of scope: auth (same as PER-20), rate limiting, metrics/logging beyond test assertions.

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
- ✓ Background orchestration: Story 2 (Tests 6–11, via `agent/src/agent/background_task.py`)
- ✓ SSE stream with `in_progress`, `completed`, `failed`: Story 2 (Tests 1–13)
- ✓ 5-second hold rule: Story 2 (Tests 6–8, 12)
- ✓ Fast and slow completed cases: Story 2 (Tests 6–8)
- ✓ Tool/LLM failure: Story 2 (Tests 9–10)
- ✓ Unknown run id: Story 2 (Test 4)
- ✓ Existing POST /query: Story 3 (Tests 1–3, 5, 9)

---

## Notes & Open Questions

**Assumptions:**
- `process_question(query: str) → dict` signature and return type are stable per PER-20 and verified in Story 3 (AC-6).
- UUID v4 collision probability is negligible; no locking required for id generation.
- In-memory run store is acceptable for proof-of-concept; multi-process deployments will upgrade to Redis.
- 1-hour TTL for run cleanup is reasonable; configurable via env var if needed.
- All routes registered in `agent/src/main.py` per change_requests.md directive.

**Deferred:**
- Run history API (`GET /agent/runs`) — not in scope.
- Auth/authorization — out of scope per requirements.
- Metrics/logging beyond startup logs — out of scope.
- Rate limiting — assume upstream FastAPI middleware.
```
