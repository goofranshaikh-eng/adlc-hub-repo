FAIL

**Concrete mismatches:**

1. **File paths in breakdown.md and stories conflict with change_requests.md:**
   - breakdown.md specifies: `agent/src/agent/routes.py`, `agent/src/routes/agent.py`
   - change_requests.md directive: "Do not add agent/src/agent/routes.py or agent/src/routes/agent.py. Use that one path in the specs, breakdown, and stories."
   - All route handlers must be in `agent/src/main.py` per change_requests.md, but breakdown.md and stories still reference separate routes files.

2. **stories/01-agent.md, 02-agent.md, 03-agent.md file paths inconsistent:**
   - stories/01-agent.md specifies new file `agent/src/agent/models.py` (OK).
   - stories/01-agent.md specifies modified file `agent/src/main.py` with POST /agent/query handler (OK).
   - But stories/02-agent.md specifies new file `agent/src/agent/sse_handler.py` and `agent/src/agent/background_task.py` with GET route handler in `agent/src/main.py` (OK).
   - However, the interface in stories/02-agent.md shows background_task spawn signature includes `run_store: "RunStore"` parameter, but stories/01-agent.md shows spawn signature takes only `(agent_run_id, query)` without run_store—**AC mismatch on function signature**.

3. **Run store method implementation inconsistency:**
   - stories/01-agent.md line "File Plan" shows `agent/src/agent/run_store.py` created in Story 1.
   - stories/02-agent.md shows `agent/src/agent/run_store.py` extended in Story 2 with `enqueue_event()` and `cleanup_expired()`.
   - But specs/1.md shows run_store.enqueue_event() and mark_failed() called in Story 1 acceptance path, yet these methods are not implemented in the Story 1 interfaces—**they are deferred to Story 2, violating AC-1 and AC-2 verification** (Events must be enqueued synchronously on rejection before POST returns, per AC-2 spec).

4. **Acceptance Criteria mapping in breakdown differs from test count:**
   - breakdown.md states AC-1 requires 2 tests, AC-2 requires 6 tests (total 8 for Story 1).
   - stories/01-agent.md Tests section lists 8 tests numbered 1–8, matching the count (OK).
   - But Test 8 in stories/01-agent.md verifies "agent_run_id consistent in SSE stream"—this is a Story 2 concern (SSE delivery), not Story 1 (immediate accept). **AC-1 and AC-2 do not require SSE stream verification; that is AC-3 scope.**

5. **Change_requests.md directive not applied to specs files:**
   - specs/1.md, specs/2.md, specs/3.md still reference `agent/src/routes/agent.py` and separate routes modules.
   - Should reference only `agent/src/main.py` per change_requests.md.
   - specs/PER-43.md, specs/PER-44.md, specs/PER-45.md also reference separate routes files or do not lock routes to main.py.

6. **Pydantic model validation mismatch:**
   - specs/1.md and stories/01-agent.md define `AgentQueryRequest(BaseModel)` with `query: str = Field(..., min_length=1)`.
   - FastAPI will auto-reject missing or non-string `query` at HTTP 422 (Pydantic validation layer).
   - But AC-1 and AC-2 require HTTP 200 responses, never 422.
   - **Solution required:** Custom exception handler or wrapper to convert 422 to 200 rejected response, but this is not documented in any spec or story. **Incomplete implementation contract.**

7. **Background task spawn in Story 1 incomplete:**
   - stories/01-agent.md shows `asyncio.create_task(spawn_background_job(run.id, query, run_store))` call in POST handler.
   - But `spawn_background_job()` in stories/01-agent.md background_task.py interface shows `async def spawn_background_job(agent_run_id: str, query: str, run_store: "RunStore") -> None: pass` (placeholder, no-op).
   - **This violates Story 1 definition:** Story 1 must spawn the task; Story 2 implements orchestration logic. Current breakdown defers the **entire spawn call** to Story 2 by leaving spawn_background_job() empty, but spawn itself must happen in Story 1 to satisfy "launch background job asynchronously" in AC-1.

8. **Test coverage for Story 1 Tests 6 not verifiable in Story 1 alone:**
   - Test 6: "Rejection does not call `process_question`" — requires mock/spy on `process_question`.
   - But `process_question` is in `agent/src/agent/orchestrator.py` (per change_requests.md preserved modules).
   - Story 1 does not import or call `process_question` in its handler; spawn_background_job() is a no-op placeholder.
   - **This test cannot fail in Story 1** because rejection path calls `run_store.mark_failed()`, not `spawn_background_job()`. The test is artifactually passing due to incomplete implementation.

9. **Missing merged_plan.md:**
   - Ticket specifies "check…merged_plan.md when present."
   - No merged_plan.md file provided in context.
   - Cannot verify story execution order gaps or merged story consolidation logic.
   - **Cannot fully verify breakdown story dependencies and sequencing without this file.**

10. **Specs file for PER-43, PER-44, PER-45 missing schema lock on request/response models:**
    - specs/PER-43.md, specs/PER-44.md, specs/PER-45.md all define Pydantic models.
    - But no specs/PER-43.md file present in the file list (only specs/1.md, specs/2.md, specs/3.md).
    - Inconsistency: `proposed_tickets.md` maps `01-agent` → `PER-43`, etc., but specs directory has numeric files, not ticket-id files.
    - **File path inconsistency:** Specs should be named `specs/PER-43.md`, `specs/PER-44.md`, `specs/PER-45.md` per ADLC pattern, or numeric files should map to ticket IDs in a manifest. Current state mixes both conventions.

11. **AC-2 detailed verification incomplete:**
    - AC-2 requires: "GET /agent/runs/{agent_run_id}/events sends `failed` with that reason and does not send `completed`."
    - stories/02-agent.md Test 11 verifies this (invalid query → failed, no completed).
    - But stories/01-agent.md does not verify that the `failed` event is actually enqueued and retrievable via SSE in Story 1 tests.
    - Test 8 in stories/01-agent.md is titled "agent_run_id consistent in SSE stream" but SSE is Story 2 scope—**Test 8 should not exist in Story 1 tests; it belongs in Story 2 verification.**
