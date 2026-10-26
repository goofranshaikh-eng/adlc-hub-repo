```markdown
# Story 03-agent: Preserve existing POST /query and GET /health endpoints

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
├── src/main.py                         (audit existing POST /query and GET /health; no changes)
└── src/agent/config.py                 (if exists; verify MCP_URL unchanged)
```

---

## Interfaces

### POST /query (existing — no changes)

**Request:**
```json
{
  "question": "<non-empty string>"
}
```

**Response (success):**
```json
{
  "answer": "<text>"
}
```

**Response (error):**
```json
{
  "error": "<error message>"
}
```

**Behavior Contract:**
- Blocking: handler calls `process_question(question)` synchronously (or awaits it).
- Returns HTTP 200 immediately upon `process_question` completion.
- No changes to signature, latency profile, error handling, or tool names.

### GET /health (existing — no changes)

**Request:**
```
GET /health
Accept: application/json
```

**Response:**
```json
{
  "status": "ok"
}
```

**Behavior Contract:**
- Immediate response (no blocking).
- No changes to output format or status code.

---

## Algorithm Notes

### Verification Approach

This story performs an **audit** of existing endpoints and **regression testing** to confirm Stories 1–2 did not introduce side effects.

#### Audit Checklist

1. **POST /query handler location:**
   - Typically in `src/main.py` or `src/routes/query.py`.
   - Verify handler calls `process_question(question)` directly (blocking).
   - Confirm no async/await wrapper that alters behavior.

2. **GET /health handler location:**
   - Typically in `src/main.py` or `src/routes/health.py`.
   - Verify handler returns `{"status": "ok"}`.
   - Confirm no dependency on run store or new agent routes.

3. **Tool names and MCP integration:**
   - Search codebase for tool name definitions (if any).
   - Search for `MCP_URL` or MCP configuration.
   - Confirm these are unchanged from pre-Story-1 state.

4. **ID normalization:**
   - If any query/question ID normalization exists (e.g., slugification), verify it is unchanged.
   - No new normalization logic introduced.

5. **Process isolation:**
   - Verify `process_question` is called only from the main agent process.
   - No child processes or thread pools spawned for MCP calls.
   - Confirm Stories 1–2 do not fork or thread.

#### Regression Test Categories

**Category A: Blocking Behavior**
- POST /query with valid question blocks until `process_question` completes.
- Measure latency; confirm no regression compared to baseline.

**Category B: Response Format**
- POST /query returns `{"answer": "..."}` or `{"error": "..."}` as JSON.
- GET /health returns `{"status": "ok"}`.

**Category C: Coexistence**
- Both POST /agent/query (new, non-blocking) and POST /query (existing, blocking) work in same session.
- No global state corruption or route collision.

**Category D: MCP Isolation**
- MCP is called only from main process (no child processes).
- Tool names and configuration unchanged.

---

## Verify

### Test File: `agent/tests/test_backward_compat.py`

#### Test 1: AC-6 — POST /query with valid question returns blocking answer

```python
async def test_post_query_valid_question_returns_answer():
    """Given a valid question, POST /query returns answer after process_question completes."""
    # Setup: mock process_question to return {"answer": "The answer is 42"}
    # When: POST /query with {"question": "What is the ultimate answer?"}
    # Then: response status 200, body {"answer": "The answer is 42"}
    # And: response latency >= process_question latency (blocking)
```

**Verification:**
- Response status is 200.
- Response body contains `"answer": "<text>"`.
- Response time includes full `process_question` execution time (blocking).

#### Test 2: AC-6 — POST /query with error condition returns error blocking

```python
async def test_post_query_error_returns_error():
    """Given a question that causes error, POST /query returns error after process_question completes."""
    # Setup: mock process_question to return {"error": "MCP unavailable"}
    # When: POST /query with {"question": "Why did it fail?"}
    # Then: response status 200, body {"error": "MCP unavailable"}
    # And: response latency >= process_question latency (blocking)
```

**Verification:**
- Response status is 200.
- Response body contains `"error": "<message>"`.
- Response time includes full `process_question` execution time (blocking).

#### Test 3: AC-6 — POST /query latency unchanged (no regression)

```python
def test_post_query_latency_no_regression():
    """Given POST /query, latency is unchanged from baseline (no regression from Stories 1–2)."""
    # Setup: establish baseline latency by running N=10 POST /query calls before Stories 1–2
    # When: POST /query is called N=10 times after Stories 1–2 implementation
    # Then: mean latency after <= mean latency before + 10% tolerance
    # (Allows for normal variance; fails if Stories 1–2 add overhead)
```

**Verification:**
- Measure wall-clock time for each request.
- Calculate mean latency for pre-implementation baseline.
- Calculate mean latency for post-implementation.
- Assert `mean_post <= mean_baseline * 1.1` (10% tolerance).

#### Test 4: AC-6 — GET /health returns ok

```python
async def test_get_health_returns_ok():
    """Given GET /health, response is always ok with no side effects."""
    # When: GET /health is called
    # Then: response status 200, body {"status": "ok"}
```

**Verification:**
- Response status is 200.
- Response body is exactly `{"status": "ok"}`.

#### Test 5: AC-6 — POST /query and POST /agent/query coexist without interference

```python
async def test_post_query_and_agent_query_coexist():
    """Given both endpoints, they work independently in same session."""
    # Setup: mock process_question to return {"answer": "Hello"}
    # When: 
    #   1. POST /agent/query with {"query": "agent query"}
    #   2. Capture agent_run_id
    #   3. POST /query with {"question": "legacy query"}
    #   4. GET /agent/runs/{agent_run_id}/events
    # Then:
    #   1. Agent endpoint returns ok immediately
    #   2. Legacy endpoint returns answer blocking
    #   3. SSE stream delivers events without corruption
```

**Verification:**
- Agent POST returns immediately (< 100ms) with `agent_run_id`.
- Query POST blocks and returns answer.
- SSE stream delivers events in order; no interference.
- No exception or error in any response.

#### Test 6: AC-6 — MCP called only from main process (no child processes)

```python
def test_mcp_called_from_main_process_only():
    """Given the agent process, MCP calls originate only from main process."""
    # Setup: instrument process_question or MCP HTTP client to log process ID
    # When: POST /query is called (legacy blocking endpoint)
    # Then: all MCP HTTP calls originate from the main agent process (same PID as FastAPI)
    # And: no child processes forked or threads spawned for MCP
```

**Verification:**
- Hook `httpx` client or `process_question` to capture calling process ID (e.g., `os.getpid()`).
- Confirm process ID matches main FastAPI process ID.
- No child processes appear in `ps` or system process list during MCP call.

#### Test 7: AC-6 — Tool names used in POST /query remain unchanged

```python
def test_tool_names_unchanged():
    """Given the existing tools, their names are unchanged from pre-Stories 1–2."""
    # Setup: load tool definitions (if any) from config or agent code
    # When: audit tool registry
    # Then: tool names match pre-implementation baseline (e.g, ["web_search", "calculator"])
    # And: no new or renamed tools
```

**Verification:**
- Extract tool names from configuration or agent code.
- Compare against known baseline (document in test or external fixture).
- Assert no additions, removals, or renames.

#### Test 8: AC-6 — ID normalization logic unchanged

```python
def test_id_normalization_unchanged():
    """Given any ID normalization in agent (e.g., slugify, normalize), it is unchanged."""
    # Setup: if ID normalization exists, capture logic and baseline outputs
    # When: audit normalization code and test outputs
    # Then: normalization behavior matches pre-implementation baseline
```

**Verification:**
- If normalization exists (e.g., lowercasing, removing special chars):
  - Test with known input; confirm output matches baseline.
  - No changes to normalization function or its inputs/outputs.
- If no normalization exists: pass trivially (no change to make).

#### Test 9: AC-6 — MCP_URL configuration unchanged

```python
def test_mcp_url_configuration_unchanged():
    """Given MCP_URL setting, it is unchanged from pre-implementation."""
    # Setup: read MCP_URL from config (env var, config file, or hardcoded)
    # When: audit configuration
    # Then: MCP_URL value matches pre-implementation baseline
    # (e.g., "http://localhost:3000" or environment variable points to same location)
```

**Verification:**
- Read MCP_URL from configuration (e.g., `os.getenv("MCP_URL")` or config file).
- Compare against known baseline.
- Assert no change.

---

## Notes

- **No code changes to existing endpoints.** This story is audit + testing only.
- **Baseline establishment:** Tests 3, 7, 8, 9 require baseline values (pre-Stories 1–2). Document these in test fixtures or environment setup.
- **Coexistence test (Test 5):** Critical; verifies no global state corruption when both blocking and non-blocking endpoints run together.
- **MCP process isolation (Test 6):** Verifies no threading or multiprocessing overhead introduced.
- **Latency regression (Test 3):** 10% tolerance allows for normal variance; only fails if Stories 1–2 add significant overhead.
- **Out of scope:** Auth, rate limiting, metrics—same as AC-6 requirements.

---

## Acceptance Criteria Verification Summary

| AC | Test # | Condition | Pass Criteria |
|----|----|-----------|---------------|
| AC-6 | 1, 2, 3, 5, 9 | POST /query behavior and latency unchanged | Returns `{"answer": "…"} \| {"error": "…"}` blocking; latency no regression; MCP_URL unchanged; coexist with new endpoints |
| AC-6 | 4 | GET /health unchanged | Returns `{"status": "ok"}` |
| AC-6 | 6, 7, 8 | Process isolation and configuration unchanged | MCP calls from main process only; tool names and ID normalization unchanged |

**Total Test Count: 9**
```
