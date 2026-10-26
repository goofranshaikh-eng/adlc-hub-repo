# Story 4: End-to-end integration tests

**Repo:** `mcp` (adlc-mcp-server)  
**Story ID:** 4  
**Story Title:** Add end-to-end integration tests  
**Parent Epic:** PER-19  

---

## 1. Acceptance Criteria

**AC5:** Integration coverage for tools over HTTP / in-process FastMCP client.  
**AC1/AC2 (partial):** Invoke all four tools; validate success + error JSON.  
**AC3:** `/health` still 200 + `{"status": "ok"}`.  
**Exception 1:** GraphQL failure → tool error JSON; process stays up.

---

## 2. File Plan

### New
1. **`tests/test_integration.py`** — Tests 14–20

Mock GraphQL (httpx mock / monkeypatch `GraphQLClient.execute`). Prefer ASGI/httpx or FastMCP in-process client; full live uvicorn only if needed (Story 1 already has an SSE handshake test).

---

## 3. Tests

- 14–17: each tool over the app/MCP stack with mocked GraphQL success
- 18: GraphQL error → `{"error": "..."}`
- 19: `/health`
- 20: invalid tool input (empty id) → error shape

## 4. Definition of done

Full `pytest -q` green; lint ok; commit on `feat/PER-18`.
