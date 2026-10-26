---
key: PER-20
type: story
status: todo
blocks: []
blocked_by: []
labels: []
repos: []
summary: Integrate Agent with MCP server
---

**Job story.** When a user asks the TriNet agent a question, they want an answer from the LLM, and when that question needs sales-order or employee data, they want the agent to call the MCP tools over FastAPI and FastMCP HTTP instead of spawning a local stdio process, so the same answers work against the shared MCP server without a per-request Flask or stdio server.

**System information**

**System 1 · Agent** (`agent` / adlc-agents)

* Reads: user question on the agent HTTP API; MCP tools over FastMCP HTTP (`MCP_URL`, for example `http://127.0.0.1:8000/sse`); LLM via the existing Anthropic or gateway client.
* Writes: the answer JSON only. No GraphQL calls from the agent. No stdio MCP child process.
* Built by: ADLC on repo `agent`.
* Does not change: the four tool names and payloads on `mcp`; GraphQL ownership; when the LLM chooses a tool versus a direct answer.

**Reasoning.** The agent already decides whether to answer directly or call MCP, then asks the LLM to synthesize the tool result. This story keeps that behavior and replaces the host and the MCP transport: Flask plus a stdio subprocess becomes FastAPI plus a FastMCP HTTP client pointed at the [PER-18](https://linear.app/personalgs/issue/PER-18/serve-mcp-graphql-tools-over-fastapi-fastmcp-http-replace-stdio) MCP server.

**Acceptance criteria**

1. Given the agent is running, when a caller sends a question that does not need data (a greeting or a help request), then the LLM answers directly and no MCP tool is called.
2. Given `MCP_URL` points at the FastAPI and FastMCP server, when the LLM requests employee or sales-order data, then the agent calls that tool over FastMCP HTTP (not stdio) and returns the synthesized answer. The tools stay `list_sales_order`, `list_employees`, `get_sales_order_by_id`, and `get_employee_by_id`.
3. Given a tool returns `{"error": "..."}` or the MCP server is unreachable, when the question needed that tool, then the agent returns a clear error and stays up. It does not start a stdio MCP process.
4. Given the agent process is started, when an operator checks health on the FastAPI app, then they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool.
5. Given the `adlc-agents` README, when a developer follows setup, then they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question. HTTP MCP is the documented primary path. Flask and stdio are not.

**Exception criteria**

* The LLM or the gateway is down, so the request fails with an error and the process stays up.
* The MCP HTTP server is down or a tool returns `{"error": "..."}`, so the agent surfaces that failure and does not fall back to stdio.
* An `id` argument is still normalized the way it is today (`emp-001` becomes `EMP-001`) before the HTTP tool call.
* Auth on the agent or MCP URL is out of scope for v1 unless org policy already requires it.

**Definition of done.** The `agent` serves its query API on FastAPI. MCP access is a FastMCP HTTP client. The LLM is still called for direct answers and for synthesis after tool use. README primary path is FastAPI plus `MCP_URL`. Tests cover direct answer, one tool call over HTTP, MCP or tool error, and health. No stdio MCP subprocess remains on that path.
