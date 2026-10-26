```json
[
  {
    "path": "agent/mcp_client.py",
    "line": 24,
    "body": "**AC-2 violation: Wrong MCP transport protocol.** The code sends a plain HTTP POST with JSON-RPC payload to `MCP_URL`, but AC-2 and HLD specify **FastMCP HTTP (SSE)** transport. FastMCP uses Server-Sent Events for streaming tool calls, not plain JSON-RPC POST. The current implementation will fail against a real FastMCP server expecting SSE connection negotiation. Must use a FastMCP client library or implement SSE handshake per FastMCP spec."
  },
  {
    "path": "agent/mcp_client.py",
    "line": 1,
    "body": "**Missing FastMCP client dependency.** Requirements AC-2 and HLD ADR-2 mandate FastMCP HTTP transport, but `requirements.txt` only lists `httpx` (generic HTTP client). The repo needs `fastmcp` or `mcp` package to handle FastMCP SSE protocol correctly. Without it, the hand-rolled JSON-RPC POST in `call_tool` is not FastMCP-compliant."
  },
  {
    "path": "agent/llm_client.py",
    "line": 33,
    "body": "**AC-2 gap: Tool inference logic incomplete.** `_infer_tool` only handles `list_sales_order` and `list_employees`, but AC-2 requires **four tools**: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`. Questions like \"get employee emp-001\" will incorrectly route to `list_employees` instead of `get_employee_by_id`, and ID arguments will be lost. Must parse questions for \"get <entity> <id>\" patterns and extract the ID into `tool_args`."
  },
  {
    "path": "agent/llm_client.py",
    "line": 18,
    "body": "**AC-1/AC-2 risk: No actual LLM tool-use API.** The code calls Claude's plain `messages.create` without tools schema, then uses heuristics (`_is_greeting_or_help`, `_infer_tool`) to guess intent. AC-1 and AC-2 require the **LLM to decide** whether to call a tool. Per HLD \"LLM Client\" responsibilities, the LLM must see MCP tool schemas and request tools via Claude's tool-use API. Current implementation cannot distinguish \"list sales orders\" (data question) from \"what is a sales order?\" (help question) reliably without LLM tool-choice."
  },
  {
    "path": "tests/test_tool_call.py",
    "line": 28,
    "body": "**Test gap: AC-2 not verified.** Test mocks `httpx.post` and asserts JSON-RPC payload structure, but does not verify FastMCP SSE transport per AC-2. The mocked HTTP response will pass even though real FastMCP server would reject this protocol. Need integration test or explicit FastMCP client mock."
  },
  {
    "path": "tests/test_id_normalization.py",
    "line": 31,
    "body": "**Test gap: AC-2 tool coverage incomplete.** Test only covers `get_employee_by_id`. AC-2 specifies **four tools** (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`). Need tests for `get_sales_order_by_id` with ID normalization (`so-123` → `SO-123`) and the two list tools."
  },
  {
    "path": "agent/orchestrator.py",
    "line": 18,
    "body": "**AC-3 gap: No error surfacing on tool failure.** When `tool_result` contains `{\"error\": \"...\"}`, the code passes it to LLM for synthesis, but AC-3 requires \"the agent returns a clear error\". If the LLM synthesizes \"I'm sorry, the employee was not found\" from a tool error, that's synthesis, not a clear error return per AC-3. Should check `tool_result.get(\"error\")` and return `{\"error\": tool_result[\"error\"]}` directly, or clarify whether synthesis counts as \"clear error\"."
  },
  {
    "path": "README.md",
    "line": 97,
    "body": "**AC-5 process note: Example questions claim incorrect behavior.** README line 96 says `\"get employee emp-001\"` calls `get_employee_by_id` tool, but `agent/llm_client.py` `_infer_tool` only checks for \"employee\" substring and routes to `list_employees`. The documented contract does not match the implementation. Either fix `_infer_tool` to handle `get <entity> <id>` or update README examples."
  },
  {
    "path": "agent/health.py",
    "line": 30,
    "body": "**AC-4 clarification: Health probe method mismatch.** Code uses `client.head(url)` to probe `MCP_URL`, but FastMCP SSE endpoints typically do not respond to HEAD (they expect GET with SSE headers). Probe may incorrectly report `mcp_reachable: false` for a working FastMCP server. Should use GET with Accept: text/event-stream or match the connectivity check that `mcp_client` will use."
  },
  {
    "path": "agent/mcp_client.py",
    "line": 52,
    "body": "**Exception criterion violation: ID normalization incomplete.** `_normalize_id` only handles `prefix-digits` pattern (e.g. `emp-001`), but does not normalize IDs that lack a hyphen (e.g. `emp001`) or have different separators. If a user asks \"get employee emp001\", the ID is passed unchanged. Exception criterion says \"ID normalization... the way it is today\"; if existing logic handled more cases, this is a regression. Verify old normalization rules or document supported format."
  }
]
```
