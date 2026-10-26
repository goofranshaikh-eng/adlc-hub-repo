# Story 2: GraphQL client with error handling

**Repo:** `mcp` (adlc-mcp-server)  
**Story ID:** 2  
**Story Title:** Implement GraphQL client with error handling  
**Parent Epic:** PER-19  
**Root:** Work in the ADLC mcp worktree for PER-18 (same as Story 1). Do not create a new repo.

---

## 1. Acceptance Criteria

**AC2 (partial - error handling):**  
Given GraphQL returns an error or is unreachable, when the client is called, then it raises an exception with a descriptive message.

**Exception 1 (GraphQL down):**  
Given GraphQL is down or returns errors, when a query is executed, then the client raises an exception (tool layer will format `{"error": "..."}` in Story 3).

Story 1 already fails fast if `GRAPHQL_ENDPOINT` is missing — keep that. This story adds a reusable async client and unit tests (Tests 4–7 from requirements).

---

## 2. File Plan

### New
1. **`graphql_client.py`** — async GraphQL executor using `httpx.AsyncClient`
2. **`tests/test_graphql_client.py`** — Tests 4–7 (mock httpx; no real network)

### Update
1. **`server.py`** — keep startup validation; optionally construct/share a client factory used later by tools

---

## 3. Interfaces

```python
class GraphQLClient:
    def __init__(self, endpoint: str | None = None): ...
    async def execute(self, query: str, variables: dict | None = None) -> dict: ...
```

- Raise on missing/empty endpoint, HTTP errors, network errors, or GraphQL `errors` in JSON.
- Message examples: `"GraphQL endpoint unreachable"`, `"GraphQL returned error: ..."`.

---

## 4. Tests (must pass)

- Test 4: successful query returns `data`
- Test 5: network error → exception
- Test 6: GraphQL `errors` field → exception
- Test 7: non-2xx HTTP → exception

## 5. Out of scope

- MCP tools (Story 3)
- README (Story 5)

## 6. Definition of done

`pytest` for `tests/test_graphql_client.py` passes; lint (`compileall`) passes; commit on `feat/PER-18`.
