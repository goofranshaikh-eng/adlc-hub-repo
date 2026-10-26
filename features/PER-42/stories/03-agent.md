# Story LLD: 03-agent — Preserve existing POST /query and GET /health endpoints

**Ticket ID:** PER-45  
**Story ID:** 03-agent  
**Repository:** `agent`  
**Acceptance Criteria:** AC-6  
**Story Type:** Backend Verification & Testing  

---

## Acceptance Criteria

### AC-6: Existing /query path unchanged

**Given** the existing `POST /query` endpoint  
**When** it receives `{"question": "..."}` and `GET /health` is called  
**Then** `POST /query` still returns the blocking `{"answer": "..."}` or `{"error": "..."}`, `GET /health` is unchanged, and MCP is still called only from the agent process.

---

## File Plan

All paths resolve from the `agent` repository root.

### New Files

```
agent/
└── tests/
    └── test_backward_compat.py
```

### Modified Files

```
agent/
└── src/main.py                         (audit existing POST /query and GET /health; no code changes)
```

### Configuration Files (Audit Only)

```
agent/
└── .env or config file                 (verify MCP_URL unchanged — audit only, no modifications)
```

---

## Interfaces

### POST /query (existing — no changes)

**Request:**
```json
POST /query
Content-Type: application/json

{
  "question": "<non-empty string>"
}
```

**Response (success):**
```json
HTTP 200
Content-Type: application/json

{
  "answer": "<text>"
}
```

**Response (error):**
```json
HTTP 200
Content-Type: application/json

{
  "error": "<error message>"
}
```

**Behavior Contract:**
- Blocking: handler calls `process_question(question)` directly and awaits completion.
- Returns HTTP 200 immediately upon `process_question` completion.
- No changes to signature, latency profile, error handling, or tool names.
- MCP is called only from the main agent process (no child processes or threads).

### GET /health (existing — no changes)

**Request:**
```http
GET /health
Accept: application/json
```

**Response:**
```json
HTTP 200
Content-Type: application/json

{
  "status": "ok"
}
```

**Behavior Contract:**
- Immediate response (no blocking).
- No changes to output format or status code.
- No dependency on run store or new agent routes.

---

## Algorithm Notes

### Verification Approach

This story performs an **audit** of existing endpoints and **regression testing** to confirm Stories 1–2 did not introduce side effects or performance regressions.

#### Audit Checklist

1. **POST /query handler location:**
   - Located in `src/main.py` (per change_requests.md).
   - Verify handler calls `process_question(question)` directly (blocking).
   - Confirm no async/await wrapper that alters behavior from pre-Story-1 state.

2. **GET /health handler location:**
   - Located in `src/main.py` (per change_requests.md).
   - Verify handler returns `{"status": "ok"}`.
   - Confirm no dependency on run store, SSE streams, or new agent routes.

3. **Tool names and MCP integration:**
   - Search codebase for tool name definitions (in `agent/orchestrator.py` or config).
   - Search for `MCP_URL` configuration (environment variable or config file).
   - Confirm these are unchanged from pre-Story-1 state.

4. **ID normalization:**
   - If any query/question ID normalization exists in `process_question` or prior handlers, verify it is unchanged.
   - No new normalization logic introduced by Stories 1–2.

5. **Process isolation:**
   - Verify `process_question` is called only from the main agent process.
   - No child processes or thread pools spawned for MCP calls by Stories 1–2 background tasks.
   - Confirm Stories 1–2 use only `asyncio.create_task()` (same process, no fork/threads).

#### Regression Test Categories

**Category A: Blocking Behavior**
- POST /query with valid question blocks until `process_question` completes.
- Measure latency; confirm no regression compared to baseline.

**Category B: Response Format**
- POST /query returns `{"answer": "..."}` or `{"error": "..."}` as JSON.
- GET /health returns `{"status": "ok"}`.

**Category C: Coexistence**
- Both POST /agent/query (new, non-blocking from Stories 1–2) and POST /query (existing, blocking) work in same session.
- No global state corruption, route collision, or event queue interference.

**Category D: MCP Isolation**
- MCP is called only from main process (no child processes spawned).
- Tool names and configuration unchanged.
- No threading or multiprocessing overhead introduced.

---

## Tests

### Test File: `agent/tests/test_backward_compat.py`

#### Test 1: AC-6 — POST /query with valid question returns blocking answer

```python
async def test_post_query_valid_question_returns_answer():
    """Given a valid question, POST /query returns answer after process_question completes."""
    # Setup: create test client, mock process_question to return {"answer": "The answer is 42"}
    # When: POST /query with {"question": "What is the ultimate answer?"}
    # Then: response status 200, body {"answer": "The answer is 42"}
    # And: response is delayed by process_question latency (blocking behavior)
```

**Verification:**
- Response status is 200.
- Response body contains `"answer": "<text>"` (exact match with mock return).
- Response elapsed time >= mock `process_question` latency (confirms blocking).

#### Test 2: AC-6 — POST /query with error condition returns error blocking

```python
async def test_post_query_error_returns_error():
    """Given a question that causes error, POST /query returns error after process_question completes."""
    # Setup: mock process_question to return {"error": "MCP unavailable"}
    # When: POST /query with {"question": "Why did it fail?"}
    # Then: response status 200, body {"error": "MCP unavailable"}
    # And: response is delayed by process_question latency (blocking behavior)
```

**Verification:**
- Response status is 200.
- Response body contains `"error": "<message>"` (exact match with mock error).
- Response elapsed time >= mock `process_question` latency (confirms blocking).

#### Test 3: AC-6 — POST /query latency unchanged (no regression from Stories 1–2)

```python
def test_post_query_latency_no_regression():
    """Given POST /query, latency is unchanged from baseline (Stories 1–2 add no overhead)."""
    # Setup: establish baseline latency from pre-Story-1 state (document in fixture or constant)
    # When: POST /query is called N=10 times after Stories 1–2 implementation
    # Then: mean latency after <= mean latency before + 10% tolerance
```

**Verification:**
- Measure wall-clock time for 10 sequential POST /query calls.
- Calculate mean latency for post-Stories-1–2 implementation.
- Load baseline from fixture or test constant.
- Assert `mean_post <= mean_baseline * 1.1` (10% tolerance for JIT, system variance).

#### Test 4: AC-6 — GET /health returns ok

```python
async def test_get_health_returns_ok():
    """Given GET /health, response is always ok with no side effects from Stories 1–2."""
    # Setup: create test client
    # When: GET /health is called
    # Then: response status 200, body {"status": "ok"}
```

**Verification:**
- Response status is 200.
- Response body is exactly `{"status": "ok"}`.
- No exception or error.

#### Test 5: AC-6 — POST /query and POST /agent/query coexist without interference

```python
async def test_post_query_and_agent_query_coexist():
    """Given both endpoints in same session, they work independently."""
    # Setup: mock process_question to return {"answer": "Hello"}
    # When: 
    #   1. POST /agent/query with {"query": "agent query"}; capture agent_run_id
    #   2. POST /query with {"question": "legacy query"}; verify blocks
    #   3. GET /agent/runs/{agent_run_id}/events; open SSE and read 1 event
    # Then:
    #   1. Agent POST returns 200 immediately (< 100ms)
    #   2. Query POST returns 200 blocking with answer
    #   3. SSE delivers in_progress event (no corruption)
```

**Verification:**
- Agent POST response time < 100ms, body contains `agent_run_id`.
- Query POST response time >= mock latency, body contains `answer`.
- SSE first event is `in_progress` with correct `run_id`.
- No exceptions or global state corruption.

#### Test 6: AC-6 — MCP called only from main process (no child processes)

```python
def test_mcp_called_from_main_process_only():
    """Given the agent process, MCP calls originate only from main process."""
    # Setup: instrument process_question or httpx client to capture process ID
    # When: POST /query is called
    # Then: all MCP HTTP calls originate from main FastAPI process PID
    # And: no child processes appear during MCP call (verify via os.getpid() hook)
```

**Verification:**
- Instrument `process_question` or MCP HTTP client to log `os.getpid()` on each call.
- Verify logged PID matches the main FastAPI process PID (from test setup).
- Confirm no child processes forked (query `psutil.Process.children()` or `ps` output).

#### Test 7: AC-6 — Tool names used in POST /query remain unchanged

```python
def test_tool_names_unchanged():
    """Given existing tools, their names are unchanged from pre-Stories 1–2."""
    # Setup: load tool definitions from agent codebase (typically in orchestrator.py or config)
    # Load baseline tool names (document in test fixture or constant)
    # When: audit tool registry in current codebase
    # Then: current tool names match baseline exactly (no additions, removals, renames)
```

**Verification:**
- Extract tool names from `agent/orchestrator.py` or configuration (search for tool definitions).
- Load baseline from test fixture (e.g., `BASELINE_TOOL_NAMES = ["web_search", "calculator"]`).
- Assert current set == baseline set (exact match; no diffs).

#### Test 8: AC-6 — ID normalization logic unchanged

```python
def test_id_normalization_unchanged():
    """Given any ID normalization in agent, behavior is unchanged."""
    # Setup: if ID normalization exists in process_question or handlers, identify it
    # Load baseline test inputs/outputs (document in fixture)
    # When: apply normalization to baseline inputs in current codebase
    # Then: outputs match baseline (no changes to normalization function or logic)
```

**Verification:**
- Search codebase for ID normalization (e.g., slugify, lowercase, remove special chars).
- If found:
  - Apply to baseline test inputs (e.g., "Example Query" → expected output).
  - Confirm output matches baseline (no logic changes).
- If not found:
  - Pass trivially (no normalization to change).

#### Test 9: AC-6 — MCP_URL configuration unchanged

```python
def test_mcp_url_configuration_unchanged():
    """Given MCP_URL setting, it is unchanged from pre-Stories 1–2."""
    # Setup: read MCP_URL from config (os.getenv("MCP_URL") or config file)
    # Load baseline MCP_URL (document in test fixture or constant)
    # When: audit current MCP_URL value
    # Then: current value matches baseline (no configuration changes)
```

**Verification:**
- Read `MCP_URL` from environment (e.g., `os.getenv("MCP_URL")`) or config file.
- Load baseline from test fixture (e.g., `BASELINE_MCP_URL = "http://localhost:3000"`).
- Assert current == baseline (exact string match).

---

## Verify

**Test Count:** 9 tests across AC-6 (see Test 1–9 above).

**Story 3 (PER-45: 03-agent):** 9 tests verify existing POST /query and GET /health behavior, coexistence with new non-blocking endpoints, and PER-20 contract stability (tool names, MCP URL, process isolation).

All tests are listed in this document.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|----|-----------|---------------|
| AC-6 | 1, 2 | POST /query blocking behavior preserved | Returns `{"answer": "…"}` or `{"error": "…"}` blocking; response time includes full `process_question` latency |
| AC-6 | 3, 5, 9 | POST /query latency and configuration unchanged | Mean latency no regression (≤ 10% tolerance); MCP_URL unchanged; coexist with new endpoints without interference |
| AC-6 | 4 | GET /health unchanged | Returns `{"status": "ok"}` with no side effects |
| AC-6 | 6, 7, 8 | Process isolation and tool configuration unchanged | MCP calls from main process only; tool names and ID normalization unchanged from baseline |

**Total Test Count: 9**

---

## Notes

- **No code changes to existing endpoints.** This story is audit + testing only. All assertions confirm pre-Stories-1–2 behavior is preserved.
- **Baseline establishment:** Tests 3, 7, 8, 9 require baseline values captured from pre-Stories-1–2 state. Document baselines in test fixtures (e.g., `conftest.py` or module-level constants).
- **Coexistence test (Test 5):** Critical verification that Stories 1–2 do not corrupt global state or interfere with legacy endpoint when both are active in same session.
- **MCP process isolation (Test 6):** Verifies no threading or multiprocessing overhead introduced by background tasks (Stories 1–2 must use only `asyncio.create_task()`, not `Thread()`, `Process()`, or `ThreadPoolExecutor`).
- **Latency regression (Test 3):** 10% tolerance allows for normal variance (JIT compilation, system load); only fails if Stories 1–2 add significant overhead (e.g., global lock, sync work in request path).
- **Out of scope:** Auth (same as AC-6 and PER-20), rate limiting, metrics/logging beyond test assertions.

---

## Dependencies

- **On Stories 1–2:** This story verifies they do not break existing behavior. Run after Stories 1–2 are complete.
- **On PER-20:** Existing `process_question` contract is stable (no changes).
- **On baseline fixtures:** Tests 3, 7, 8, 9 require baseline values established before Stories 1–2 implementation. Document in test fixtures or test constants.

---

## Key Invariants (from change_requests.md)

- **Routes location:** POST /agent/query and GET /agent/runs/{id}/events are registered in `agent/src/main.py` (no separate routes file).
- **Existing endpoints:** POST /query and GET /health stay in `agent/src/main.py` with no changes to handlers.
- **Preserved modules:** `agent/src/agent/orchestrator.py`, `agent/src/agent/health.py`, `agent/src/agent/mcp_client.py` are not modified by Stories 1–2.
- **Stack:** Python 3.11+, FastAPI, asyncio (no new external dependencies beyond PER-20 baseline).
