---
key: PER-18
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Serve MCP GraphQL tools over FastAPI + FastMCP HTTP (replace stdio)
---

**Job story.**
When an agent or developer calls TriNet MCP tools for sales orders and employees, they want to reach the MCP server over HTTP using FastAPI and FastMCP instead of a local stdio process, so that the same tools stay queryable from multiple clients without spawning a per-client stdio server.

**System information**
**System 1 · MCP server (**`mcp` / adlc-mcp-server)

* Reads: GraphQL via `GRAPHQL_ENDPOINT` (same queries as today)
* Writes: none beyond existing tool GraphQL calls
* Built by: ADLC on repo `mcp`
* Does not change: GraphQL schema ownership; agent domain logic outside MCP client config

**Reasoning.**
This is a transport/hosting change with no new domain judgement: keep the four tools and JSON shapes; change how the server is run and discovered (stdio → FastAPI + FastMCP HTTP).

**Acceptance criteria**

1. Given the MCP server is running in the new mode, when a client connects via FastMCP HTTP (not stdio), then these tools remain available and behave as today: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`.
2. Given a valid `GRAPHQL_ENDPOINT`, when a tool is invoked over HTTP, then results use the same success/error JSON shapes as the current stdio server (list/get payloads or `{"error": "..."}`).
3. Given the server process is started, when an operator checks health/readiness on the HTTP app, then they can tell the process is up without calling a GraphQL tool.
4. Given the `adlc-mcp-server` README, when a developer follows setup, then they can run the server with FastAPI + FastMCP over HTTP and point a client at that URL (HTTP is the documented primary path).
5. Given automated tests for tools/client, when CI or local tests run, then tool behavior is still covered (HTTP or in-process FastMCP test client).

**Exception criteria**

* GraphQL is down or errors → tools return the existing error JSON shape; the server process stays up.
* A client is still configured for stdio → README documents migration to HTTP (prefer HTTP-only once callers can switch).
* Auth is out of scope for v1 unless org policy already requires it (document as follow-up).

**Definition of done.**
`mcp` runs on FastAPI + FastMCP HTTP; README primary path is HTTP; all four tools work through HTTP against GraphQL; tests green; no regression in tool names/payloads; clients know the MCP URL to use instead of stdio.
