# Breakdown: Integrate Agent with MCP server

## Overview

This epic replaces the agent's stdio MCP subprocess with a FastMCP HTTP client pointed at the shared MCP server. The agent preserves its existing LLM-driven behavior—direct answers when no data is needed, MCP tool calls when data is required, synthesis of tool results into final answers—while switching transport from Flask + stdio to FastAPI + HTTP.

**Epic Acceptance Criteria (from requirements.md):**

- AC-1: Given the agent is running, when a caller sends a question that does not need data (a greeting or a help request), then the LLM answers directly and no MCP tool is called.
- AC-2: Given `MCP_URL` points at the FastAPI and FastMCP server, when the LLM requests employee or sales-order data, then the agent calls that tool over FastMCP HTTP (not stdio) and returns the synthesized answer. The tools stay `list_sales_order`, `list_employees`, `get_sales_order_by_id`, and `get_employee_by_id`. ID arguments are normalized (supported format: `prefix-digits` with hyphen, e.g., `emp-001` → `EMP-001`).
- AC-3: Given a tool returns `{"error": "..."}` or the MCP server is unreachable, when the question needed that tool, then the agent returns a clear error and stays up. It does not start a stdio MCP process.
- AC-4: Given the agent process is started, when an operator checks health on the FastAPI app, then they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool.
- AC-5: Given the `adlc-agents` README, when a developer follows setup, then they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question. HTTP MCP is the documented primary path. Flask and stdio are not. ID format is documented (supported format: `prefix-digits` with hyphen).

**Definition of Done:**

- The `agent` serves its query API on FastAPI.
- MCP access is a FastMCP HTTP client.
- The LLM is still called for direct answers and for synthesis after tool use.
- README primary path is FastAPI plus `MCP_URL`, including ID format examples.
- Tests cover direct answer, one tool call over HTTP, MCP or tool error, and health.
- No stdio MCP subprocess remains on that path.

---

## Dependencies

**Upstream (blocking this epic):**

- [PER-18](https://linear.app/personalgs/issue/PER-18/serve-mcp-graphql-tools-over-fastapi-fastmcp-http-replace-stdio): MCP server must serve four tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) over FastMCP HTTP at `MCP_URL` before this epic's stories can integrate against it.

**Downstream (blocked by this epic):**

- None identified. This epic enables the agent to use the shared MCP server; future work may add auth or additional tools, but those are out of scope for v1.

**Cross-repo coordination:**

- `mcp` repo (PER-18): external dependency; no changes in this epic.
- `graphql` repo: external dependency; no changes in this epic.
- `agent` repo: all changes are confined here.

---

## Allowed Paths

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)

**Existing files:**

- `README.md`

**New files (to be created):**

- `agent/main.py` — FastAPI app entry point
- `agent/mcp_client.py` — FastMCP HTTP client module
- `agent/health.py` — Health endpoint handler
- `agent/orchestrator.py` — Orchestrator routing to LLM and MCP client (or modify existing)
- `agent/llm_client.py` — LLM client (preserve existing or create if missing)
- `tests/test_query_direct.py` — AC-1 test
- `tests/test_query_tool.py` — AC-2 test
- `tests/test_mcp_error.py` — AC-3 test
- `tests/test_health.py` — AC-4 test
- `requirements.txt` — Add `fastapi`, `uvicorn`, `httpx` (or `requests`)

**Modified files:**

- `README.md` — AC-5 documentation (ID format examples)

**Removed files:**

- Any Flask app entry point (e.g. `agent/flask_app.py`, if it exists)
- Any stdio MCP subprocess logic (e.g. `agent/stdio_mcp.py`, if it exists)

**No changes outside `agent` repo.**

---

## Stories (Execution Order)

### Story 1: Migrate agent from Flask to FastAPI and add health endpoint

**Repo:** `agent`

**Description:**

Replace the agent's Flask app with a FastAPI app serving the query API (`POST /query`) and add a health endpoint (`GET /health`) that reports process status and MCP HTTP endpoint reachability. The query endpoint accepts `{"question": "string"}` and returns `{"answer": "string"}` or `{"error": "string"}`. The health endpoint returns `{"status": "up", "mcp_reachable": true/false}` using a lightweight connectivity check (e.g. HEAD request or cached probe) to `MCP_URL` without calling a tool.

**Covers requirements:**

- AC-4 (health endpoint reports process up and MCP reachable)
- AC-5 (FastAPI is primary path)
- Partial AC-3 (process stays up on error)

**After:** None (first story; opens epic)

**Verification Approach:**

1. Start the FastAPI app on `uvicorn agent.main:app --host 0.0.0.0 --port 8001`.
2. `GET /health` returns `{"status": "up", "mcp_reachable": false}` when `MCP_URL` is unset or unreachable.
3. `POST /query` with `{"question": "hello"}` returns `{"answer": "..."}` (placeholder or error until LLM integrated in Story 2).
4. Process stays up after each request.

**Test count:** 2 tests (health endpoint when MCP unreachable; query endpoint returns response and stays up).

**Risks:**

- R-5 (async/sync HTTP client choice): Mitigated by preferring `httpx` async client for FastAPI.

**Notes:**

- Flask app and any stdio MCP logic are removed at this stage or in Story 3.
- Health probe uses `MCP_URL` env var; if unset, `mcp_reachable: false`.
- Query logic is stubbed here; Stories 2 and 3 integrate LLM and MCP client.

---

### Story 2: Integrate LLM client for direct answers and tool routing

**Repo:** `agent`

**Description:**

Integrate the existing LLM client (Anthropic or gateway) into the FastAPI query endpoint to decide whether to answer directly or request an MCP tool. When the question is a greeting or help request, the LLM answers directly and no MCP tool is called. When the LLM requests a tool, route to a (stubbed) MCP client; Story 3 implements the actual FastMCP HTTP client. The orchestrator routes the question to the LLM client, and the LLM synthesizes tool results (or direct answer) into the final response.

**Covers requirements:**

- AC-1 (direct LLM answers when no data needed)
- Partial AC-2 (LLM requests tool, synthesis; actual HTTP call in Story 3)
- Partial AC-3 (error on LLM unavailable; MCP error in Story 3)

**After:** Story 1 (FastAPI app must exist)

**Verification Approach:**

1. `POST /query` with `{"question": "hello"}` → LLM answers directly, no MCP call logged, returns `{"answer": "Hello! How can I help you?"}`.
2. `POST /query` with `{"question": "list sales orders"}` → LLM requests tool, orchestrator routes to stubbed MCP client, returns placeholder or error (actual MCP call in Story 3).
3. LLM unavailable (mock or env unset) → returns `{"error": "LLM unavailable"}`, process stays up.

**Test count:** 3 tests (direct answer, tool request routed to stub, LLM unavailable error).

**Risks:**

- R-4 (LLM decision divergence): Mitigated by preserving existing LLM choice logic; test direct answer and tool request paths.

**Notes:**

- LLM client is async-wrapped if sync (e.g. `asyncio.to_thread`).
- Tool request is routed to a stub that returns `{"error": "MCP not implemented"}` until Story 3.

---

### Story 3: Implement FastMCP HTTP client and integrate tool calls

**Repo:** `agent`

**Description:**

Implement the FastMCP HTTP client in `agent/mcp_client.py` to call the four MCP tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) over HTTP at `MCP_URL`. Normalize `id` arguments for the supported format (`prefix-digits` with hyphen, e.g., `emp-001` → `EMP-001`) before the HTTP call, preserving existing logic. Other formats (e.g., `emp001` without hyphen) are passed through as-is. Handle tool errors (`{"error": "..."}`) and MCP server unreachable by returning clear errors; do not fall back to stdio. Integrate the client into the orchestrator so the LLM's tool requests call the MCP server over HTTP, and the synthesized answer is returned. Update the health endpoint to probe `MCP_URL` and report `mcp_reachable: true` when the MCP server is up.

**Covers requirements:**

- AC-2 (tools called over FastMCP HTTP, four tool names unchanged, ID normalization for supported format)
- AC-3 (clear error on MCP failure, no stdio fallback)
- AC-4 (health endpoint reports MCP reachable when up)

**After:** Story 2 (LLM client and orchestrator must exist)

**Verification Approach:**

1. `POST /query` with `{"question": "list sales orders"}` → LLM requests `list_sales_order` → FastMCP HTTP client calls `MCP_URL` → synthesized answer returned.
2. ID normalization test: `POST /query` with `{"question": "get employee emp-001"}` → client normalizes to `EMP-001` before HTTP call, correct employee returned.
3. MCP unreachable (mock or `MCP_URL` down) → returns `{"error": "MCP server unreachable"}`, process stays up, no stdio subprocess.
4. Tool error (mock tool returns `{"error": "not found"}`) → agent returns `{"error": "not found"}` or synthesized error, stays up.
5. `GET /health` → `{"status": "up", "mcp_reachable": true}` when MCP server up, `false` when down.

**Test count:** 5 tests (tool call over HTTP, ID normalization, MCP unreachable, tool error, health endpoint when MCP up).

**Risks:**

- R-1 (MCP server unavailable): Mitigated by AC-3 clear error, AC-4 health endpoint, operational monitoring.
- R-2 (FastMCP HTTP transport mismatch): Mitigated by integration test (verification item 1); assume SSE at `MCP_URL` per PER-18.
- R-3 (ID normalization regression): Mitigated by verification item 2; only supported format (`prefix-digits` with hyphen) is normalized.

**Notes:**

- Use `httpx` async client for FastMCP HTTP calls.
- Health probe uses lightweight check (HEAD request or cached connectivity status) to avoid latency.
- Remove any Flask app and stdio MCP logic at this stage if not already removed in Story 1.
- Unsupported ID formats (e.g., `emp001` without hyphen) are passed through as-is.

---

### Story 4: Update README to document FastAPI primary path and remove Flask/stdio

**Repo:** `agent`

**Description:**

Update `README.md` to document the FastAPI primary path: install dependencies (`pip install -r requirements.txt`), set `MCP_URL` to the FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`), set LLM credentials, run `uvicorn agent.main:app --host 0.0.0.0 --port 8001`, and query with `POST http://localhost:8001/query`. Remove all Flask and stdio MCP documentation. The README must guide a developer through setup and first query end-to-end. Document the supported ID format (`prefix-digits` with hyphen, e.g., `emp-001` → `EMP-001`) with examples.

**Covers requirements:**

- AC-5 (README documents FastAPI + `MCP_URL` as primary path; Flask and stdio removed; ID format documented)

**After:** Story 3 (FastAPI app, LLM client, and FastMCP HTTP client must be complete)

**Verification Approach:**

1. Follow the updated README setup instructions from a clean environment (or CI container).
2. Set `MCP_URL` to a running MCP server (or mock).
3. Set LLM credentials.
4. Run `uvicorn agent.main:app --host 0.0.0.0 --port 8001`.
5. `POST http://localhost:8001/query` with `{"question": "list sales orders"}` returns synthesized answer.
6. No Flask or stdio references remain in README.
7. README includes example IDs (`emp-001` → `EMP-001`) to document normalization.

**Test count:** 1 end-to-end verification (README setup → successful query).

**Risks:**

- None beyond
