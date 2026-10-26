# Story 3: Four MCP tools over FastMCP

**Repo:** `mcp` (adlc-mcp-server)  
**Story ID:** 3  
**Story Title:** Implement the four MCP tools over FastMCP  
**Parent Epic:** PER-19  
**Root:** ADLC mcp worktree for PER-18. Build on Story 1–2 files already present.

---

## 1. Acceptance Criteria

**AC1:** Tools available over FastMCP HTTP/SSE:  
`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`.

**AC2:** Success payloads match list/get shapes; failures return `{"error": "..."}` (not a process crash).

**Exception 1:** When GraphQL client raises, tool returns `{"error": "..."}` and server stays up.

**AC5 (partial):** Automated tool tests (Tests 8–13).

---

## 2. File Plan

### New
1. **`tools.py`** — register the four tools on the shared `FastMCP` instance
2. **`tests/test_tools.py`** — Tests 8–13 with mocked `GraphQLClient`

### Update
1. **`server.py`** — import/register tools so `/sse` exposes them (remove “no tools yet” stub behavior)
2. Use queries consistent with TriNet MCP GraphQL tools (sales orders + employees). Prefer simple named GraphQL operations; mock in tests.

---

## 3. Tool contracts

| Tool | Inputs | Success | Error |
|------|--------|---------|-------|
| `list_sales_order` | optional filters/limit as needed | list/payload dict | `{"error": "..."}` |
| `list_employees` | optional `limit` etc. | list/payload dict | `{"error": "..."}` |
| `get_sales_order_by_id` | `id` (required, non-empty) | object/payload | `{"error": "..."}` if missing id or GraphQL fail |
| `get_employee_by_id` | `id` (required, non-empty) | object/payload | same |

Wire tools via FastMCP (`@mcp.tool` or equivalent for FastMCP 4) onto the same `mcp` object used for `http_app(transport="sse")`.

---

## 4. Tests

- Tests 8–11: each tool success with mocked GraphQL
- Test 12: `get_sales_order_by_id` missing/empty `id` → error shape
- Test 13: GraphQL client exception → `{"error": "..."}`

## 5. Definition of done

Tools registered; `pytest tests/test_tools.py` + existing tests pass; lint ok; push `feat/PER-18`.
