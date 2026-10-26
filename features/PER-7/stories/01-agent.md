# Story 01-agent LLD

**Repo**: agent  
**Type**: story  
**Epic**: [agent] Agent which can directly answer or via MCP tools  
**Status**: Draft

---

## 1. Acceptance Criteria

*(Inherited from requirements.md and breakdown.md — no additions, no changes)*

### AC1: HTTP Endpoint
**Given** the agent service is running  
**When** a POST request is sent to `/query` with body `{"query": "<natural language>"}`  
**Then** the service returns HTTP 200 with JSON body `{"answer": "<text>"}`

### AC2: Direct Answer (No Tool)
**Given** the agent receives a question requiring no data lookup (e.g., "hello", "help")  
**When** the LLM determines no MCP tool is needed  
**Then** the agent returns a direct text answer without invoking any tool

### AC3: Tool Call for Employee Lookup
**Given** the agent receives query `"what is the name of employee with emp-001"`  
**When** the LLM determines data lookup is required  
**Then** the agent invokes `get_employee_by_id` via MCP stdio and returns the employee name from GraphQL data

### AC4: All Four MCP Tools Available
**Given** the MCP server (PER-6) is running over stdio  
**When** the agent initializes  
**Then** the agent can successfully list and invoke all four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`

### AC5: ID Normalization
**Given** the agent receives a query with lowercase ID like `"emp-001"`  
**When** the GraphQL mock uses uppercase form `"EMP-001"`  
**Then** the agent normalizes the ID before calling the tool and returns correct data

### AC6: Basic Tests
**Given** the implementation is complete  
**When** tests are run  
**Then** 21 unit tests with mocked LLM/MCP pass (6 HTTP server tests, 3 LLM client tests, 4 MCP client tests, 8 orchestrator tests) AND 1 integration test with mocked services passes AND a documented manual verification script successfully demonstrates AC1–AC5 against real PER-5/PER-6 services

---

## 2. File Plan

All paths are relative to `/home/goos/projects/adlc-agents`.

### 2.1 New Files

| Path | Purpose | Depends On |
|------|---------|------------|
| `app/agent/__init__.py` | Package marker for agent module | — |
| `app/agent/http_server.py` | Flask HTTP server with `/query` and `/health` endpoints | `orchestrator.py` |
| `app/agent/orchestrator.py` | Agent orchestrator coordinating LLM and MCP | `llm_client.py`, `mcp_client.py` |
| `app/agent/llm_client.py` | Anthropic LLM client for inference and tool-calling | — |
| `app/agent/mcp_client.py` | MCP stdio client for subprocess management and tool invocation | — |
| `tests/test_http_server.py` | Unit tests for HTTP server (request validation, error handling, health check) | `app/agent/http_server.py` |
| `tests/test_orchestrator.py` | Unit tests for orchestrator (direct answer, tool-calling, ID normalization, argument validation, synthesis loop detection) | `app/agent/orchestrator.py` |
| `tests/test_llm_client.py` | Unit tests for LLM client (API calls, parsing, error handling) | `app/agent/llm_client.py` |
| `tests/test_mcp_client.py` | Unit tests for MCP client (subprocess lifecycle, tool discovery, invocation, timeout) | `app/agent/mcp_client.py` |
| `tests/test_integration.py` | Integration test with mocked PER-5/PER-6 services | All `app/agent/*.py` |
| `tests/manual_verification.md` | Manual verification script documentation | — |

### 2.2 Modified Files

| Path | Change | Reason |
|------|--------|--------|
| `README.md` | Add quickstart, environment variables, test instructions | Document agent usage and setup |
| `requirements.txt` | Add `flask==3.1.0`, `anthropic==0.18.0` | New dependencies for HTTP server and LLM client |

---

## 3. Interfaces

### 3.1 HTTP Server (`app/agent/http_server.py`)

**Module Exports**:
```python
def create_app(orchestrator: Orchestrator) -> Flask:
    """Create and configure Flask app with /query and /health endpoints."""
    ...

def run_server(host: str = "0.0.0.0", port: int = 5000) -> None:
    """Start the HTTP server (blocking)."""
    ...
```

**Endpoint Contracts**:

**POST /query**
- Request: `{"query": "<string>"}`
- Success Response (200): `{"answer": "<string>"}`
- Client Error (400): `{"error": "<validation message>"}` (malformed JSON, missing `query`, or empty `query` after strip)
- Server Error (500): `{"error": "internal error"}` (LLM/MCP/orchestrator exception, logged)

**GET /health**
- Success Response (200): `{"status": "ok", "mcp_alive": true}`
- Degraded Response (503): `{"status": "degraded", "mcp_alive": false}` (MCP subprocess crashed)

**Dependencies**: `Orchestrator` (injected at app creation)

**Validation Notes** (fixes PR comment 4280870009):
- Check `payload.get("query")` exists and is non-empty after `strip()`.
- Reject empty string queries with 400 to prevent degenerate answers.

---

### 3.2 Orchestrator (`app/agent/orchestrator.py`)

**Class Interface**:
```python
class Orchestrator:
    def __init__(self, llm_client: LLMClient, mcp_client: MCPClient):
        """Initialize orchestrator with LLM and MCP clients."""
        ...

    def handle_query(self, query: str) -> str:
        """
        Process user query via LLM and optionally MCP tools.
        Returns final answer string.
        Raises ValueError on argument validation failures (caught by HTTP server → 400).
        Raises RuntimeError on synthesis loop detection, empty synthesis response (caught by HTTP server → 500).
        Raises exceptions on LLM/MCP failures (caught by HTTP server → 500).
        """
        ...
```

**Algorithm Outline** (see §4 for detail):
1. Construct messages: system prompt (role definition + tool schemas from MCP client) + user query.
2. Call `llm_client.infer(messages, tools=mcp_tools)` → parse response.
3. If direct text response: validate non-empty content (raise `RuntimeError` if empty), return answer.
4. If tool calls: for each call, validate arguments against tool schema (raise `ValueError` on missing required field or unknown tool), normalize IDs (`.upper()`), invoke `mcp_client.call_tool(name, args)`, collect results.
5. Append tool results to messages, call `llm_client.infer(messages)` again (synthesis step, no tools).
6. If synthesis response requests tools again: raise `RuntimeError` (synthesis loop detected).
7. If synthesis response is empty text: raise `RuntimeError` (fixes PR comment 4280870047).
8. Return synthesized answer.
9. Limit to 2 LLM calls total (initial + synthesis).

**Dependencies**: `LLMClient`, `MCPClient`

---

### 3.3 LLM Client (`app/agent/llm_client.py`)

**Class Interface**:
```python
class LLMClient:
    def __init__(self, api_key: str, base_url: str = "https://api.anthropic.com", model: str = "claude-3-5-sonnet-20241022"):
        """Initialize Anthropic client with API credentials."""
        ...

    def infer(self, messages: list[dict], tools: list[dict] | None = None) -> dict:
        """
        Call Anthropic Messages API.
        Args:
            messages: Conversation history (system, user, assistant, tool_result).
            tools: Tool schemas in Anthropic format (optional).
        Returns:
            Parsed response:
                {"type": "text", "content": "<answer>"}  (content may be empty string)
                OR
                {"type": "tool_use", "calls": [{"id": "<id>", "name": "<name>", "input": {...}}, ...]}
        Raises:
            RuntimeError on API errors (network, auth, rate limit) or if response contains no text or tool calls.
        """
        ...
```

**Tool Schema Format** (Anthropic Messages API):
```json
{
  "name": "get_employee_by_id",
  "description": "Fetch employee details by ID",
  "input_schema": {
    "type": "object",
    "properties": {
      "id": {"type": "string", "description": "Employee ID (e.g., EMP-001)"}
    },
    "required": ["id"]
  }
}
```

**Dependencies**: `anthropic` SDK (from `requirements.txt`)

**Environment Variable Handling** (fixes PR comment 4280870021):
- `base_url` and `model` both treat empty string same as unset (default fallback).
- Pattern: `os.environ.get("VAR", "default").strip() or "default"`.

**Empty Response Handling** (addresses PR comment 4280870055):
- If all text blocks concatenate to empty string, raise `RuntimeError("LLM response contained no text or tool calls")` instead of returning `{"type": "text", "content": ""}`.
- Ensures orchestrator never receives empty direct answer.

---

### 3.4 MCP Client (`app/agent/mcp_client.py`)

**Class Interface**:
```python
class MCPClient:
    def __init__(self, server_command: str, working_dir: str, env: dict[str, str]):
        """
        Initialize MCP client.
        Args:
            server_command: Command to start MCP server (e.g., "python -m mcp.server").
            working_dir: Working directory for subprocess.
            env: Environment variables for subprocess (must include GRAPHQL_ENDPOINT).
        """
        ...

    def start(self) -> None:
        """
        Spawn MCP subprocess, perform JSON-RPC handshake, fetch tool list.
        Raises exception on handshake failure (agent fails to start).
        """
        ...

    def get_tools(self) -> list[dict]:
        """
        Return list of available tool schemas (Anthropic format).
        Must be called after start().
        """
        ...

    def call_tool(self, name: str, arguments: dict, timeout: int = 10) -> dict:
        """
        Invoke MCP tool via JSON-RPC over stdio.
        Args:
            name: Tool name (e.g., "get_employee_by_id").
            arguments: Tool input (e.g., {"id": "EMP-001"}).
            timeout: Request timeout in seconds.
        Returns:
            Tool result dict (content varies by tool).
        Raises:
            TimeoutError if request exceeds timeout.
            RuntimeError on MCP protocol error, tool execution error, or excessive empty lines (CPU burn mitigation).
        """
        ...

    def is_alive(self) -> bool:
        """Check if MCP subprocess is still running."""
        ...

    def shutdown(self) -> None:
        """Gracefully terminate MCP subprocess (send EOF, wait 5s, SIGKILL if needed)."""
        ...
```

**JSON-RPC Protocol Example**:

*Request (to subprocess stdin):*
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_employee_by_id",
    "arguments": {"id": "EMP-001"}
  }
}
```

*Response (from subprocess stdout):*
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"name\": \"Alice\", \"id\": \"EMP-001\"}"
      }
    ]
  }
}
```

**Dependencies**: `subprocess` (stdlib), `json` (stdlib), `select` (stdlib, POSIX timeout enforcement), `time` (stdlib)

**CPU Burn Mitigation** (fixes PR comment 4280870070):
- Track consecutive empty lines in `_read_json` loop.
- Raise `RuntimeError` after 100 consecutive empty lines to prevent tight timeout loop.

---

## 4. Algorithm Details

### 4.1 Orchestrator Query Handling

**Function**: `Orchestrator.handle_query(query: str) -> str`

**Steps**:
1. **Construct Initial Messages**:
   ```python
   system_prompt = {
       "role": "system",
       "content": (
           "You are a helpful assistant with access to MCP tools for employee and sales order data. "
           "Use tools when the user asks for specific data. Answer directly for greetings or help requests."
       )
   }
   messages = [
       system_prompt,
       {"role": "user", "content": query}
   ]
   ```

2. **Initial LLM Inference**:
   ```python
   tools = self.mcp_client.get_tools()
   response = self.llm_client.infer(messages, tools=tools)
   llm_call_count = 1
   ```

3. **Parse Response Type**:
   - If `response["type"] == "text"`:
     ```python
     if not response.get("content", "").strip():
         raise RuntimeError("LLM direct answer returned empty text")
     return response["content"]
     ```
     (Direct answer path, AC2, fixes PR comment 4280870047)
   - If `response["type"] == "tool_use"`: proceed to step 4.

4. **Execute Tool Calls** (AC3, AC4, AC5):
   ```python
   tool_results = []
   for call in response["calls"]:
       # Validate tool name against available tools
       tool_schema = next((t for t in tools if t["name"] == call["name"]), None)
       if not tool_schema:
           raise ValueError(f"LLM requested unknown tool: {call['name']}")
       
       # Validate required fields present in tool call input
       required = tool_schema["input_schema"].get("required", [])
       for field in required:
           if field not in call["input"]:
               raise ValueError(f"Missing required field '{field}' for tool '{call['name']}'")
       
       # Normalize IDs to uppercase (AC5, AD-3)
       normalized_input = {k: v.upper() if k == "id" and isinstance(v, str) else v for k, v in call["input"].items()}
       
       # Invoke tool with 10s timeout
       result = self.mcp_client.call_tool(call["name"], normalized_input, timeout=10)
       tool_results.append({
           "tool_use_id": call["id"],
           "content": json.dumps(result)
       })
   ```

5. **Append Tool Results to Messages**:
   ```python
   messages.append({"role": "assistant", "content": response["calls"]})  # LLM's tool call
   messages.append({"role": "user", "content": tool_results})  # Tool results
   ```

6. **Synthesis LLM Call** (AD-4):
   ```python
   synthesis_response = self.llm_client.infer(messages, tools=None)  # No tools on synthesis
   llm_call_count += 1
   
   if synthesis_response["type"] != "text":
       raise RuntimeError("LLM requested tools again on synthesis step (synthesis loop detected)")
   
   if not synthesis_response.get("content", "").strip():
       raise RuntimeError("LLM synthesis returned empty text")
   
   return synthesis_response["content"]
   ```

**Error Handling**:
- `ValueError` from validation → propagate to HTTP server → 400 response
- `RuntimeError` from synthesis loop or empty response → propagate to HTTP server → 500 response
- `TimeoutError` from MCP client → propagate → 500 response
- Any other exception → propagate → 500 response

**LLM Call Count Invariant**: Exactly 2 calls on tool path (initial + synthesis), exactly 1 call on direct answer path.

---

### 4.2 MCP Client Startup

**Function**: `MCPClient.start() -> None`

**Steps**:
1. **Spawn Subprocess**:
   ```python
   self.process = subprocess.Popen(
       self.server_command.split(),
       stdin=subprocess.PIPE,
       stdout=subprocess.PIPE,
       stderr=subprocess.PIPE,
       cwd=self.working_dir,
       env=self.env,
       text=True,
       bufsize=1
   )
   ```

2. **Send Handshake Request**:
   ```json
   {
     "jsonrpc": "2.0",
     "id": 0,
     "method": "initialize",
     "params": {
       "protocolVersion": "2024-11-05",
       "capabilities": {}
     }
   }
   ```
   Write to `self.process.stdin`, flush.

3. **Read Handshake Response** (with 5s timeout):
   ```python
   response = self._read_json(timeout=5)
   if "error" in response:
       raise RuntimeError(f"MCP handshake failed: {response['error']}")
   ```

4. **Fetch Tool List**:
   ```json
   {
     "jsonrpc": "2.0",
     "id": 1,
     "method": "tools/list",
     "params": {}
   }
   ```
   Write, flush, read response (with 5s timeout), parse `result["tools"]` array.

5. **Validate Four Tools Present** (AC4):
   ```python
   expected_tools = {"list_sales_order", "list_employees", "get_sales_order_by_id", "get_employee_by_id"}
   actual_tools = {tool["name"] for tool in tools}
   if not expected_tools.issubset(actual_tools):
       raise RuntimeError(f"Missing required tools: {expected_tools - actual_tools}")
   ```

6. **Store Tools**:
   ```python
   self.tools = self._convert_to_anthropic_format(tools)
   ```

**Error Handling**:
- Subprocess fails to start → raise exception (agent fails to start, logged in `run_server`)
- Handshake timeout → raise `RuntimeError`
- Tool validation fails → raise `RuntimeError`

---

### 4.3 MCP Tool Invocation with Timeout

**Function**: `MCPClient.call_tool(name: str, arguments: dict, timeout: int) -> dict`

**Steps**:
1. **Build JSON-RPC Request**:
   ```python
   request_id = self._next_id()  # Internal counter for request tracking
   request = {
       "jsonrpc": "2.0",
       "id": request_id,
       "method": "tools/call",
       "params": {"name": name, "arguments": arguments}
   }
   ```

2. **Write to Subprocess and Read with Timeout**:
   ```python
   self.process.stdin.write(json.dumps(request) + "\n")
   self.process.stdin.flush()
   
   response = self._read_json(timeout=timeout)
   ```

3. **Parse Response**:
   ```python
   if "error" in response:
       raise RuntimeError(f"MCP tool '{name}' returned error: {response['error']}")
   return response["result"]
   ```

**Notes**:
- `_read_json` uses `select.select` with deadline tracking to enforce timeout.
- Windows alternative: use `threading.Timer` to interrupt blocking read after timeout.
- Tool result format varies by tool; orchestrator expects dict or parseable JSON string.

---

### 4.4 MCP Client `_read_json` Helper (fixes PR comment 4280870070)

**Function**: `MCPClient._read_json(timeout: int) -> dict`

**Steps**:
1. **Initialize Deadline and Empty Line Counter**:
   ```python
   import time
   deadline = time.monotonic() + timeout
   empty_line_count = 0
   ```

2. **Loop Until Valid JSON or Timeout**:
   ```python
   while True:
       remaining = deadline - time.monotonic()
       if remaining <= 0:
           raise TimeoutError(f"MCP read timed out after {timeout}s")
       
       ready, _, _ = select.select([self.process.stdout], [], [], remaining)
       if not ready:
           raise TimeoutError(f"MCP read timed out after {timeout}s")
       
       line = self.process.stdout.readline()
       if not line:
           raise RuntimeError("MCP subprocess closed stdout unexpectedly")
       
       line = line.strip()
       if not line:
           empty_line_count += 1
           if empty_line_count > 100:
               raise RuntimeError("MCP subprocess emitted >100 consecutive empty lines (possible keepalive loop)")
           continue  # Skip blank lines
       
       empty_line_count = 0  # Reset counter on non-empty line
       
       try:
           return json.loads(line)
       except json.JSONDecodeError:
           continue  # Skip non-JSON lines (e.g., debug output)
   ```

**Mitigation for CPU Burn**:
- Track consecutive empty lines.
- Raise `RuntimeError` after 100 consecutive empty lines to prevent tight loop near deadline.
- Reset counter on any non-empty line (valid progress).

---

## 5. Environment Variables

Document in `README.md` (updated as part of this story):

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | Yes | — | Anthropic API key for LLM client |
| `ANTHROPIC_BASE_URL` | No | `https://api.anthropic.com` | Anthropic API base URL (override for gateway); empty treated as unset |
| `ANTHROPIC_MODEL` | No | `claude-3-5-sonnet-20241022` | Anthropic model ID; empty treated as unset (fixes PR comment 4280870021) |
| `GRAPHQL_ENDPOINT` | Yes | — | GraphQL service URL for MCP subprocess (e.g., `http://localhost:4000/graphql`) |
| `MCP_SERVER_COMMAND` | No | `python -m mcp.server` | Command to start MCP server |
| `MCP_SERVER_WORKING_DIR` | Yes | — | Working directory for MCP subprocess (PER-6 repo path) |
| `FLASK_HOST` | No | `0.0.0.0` | HTTP server bind host |
| `FLASK_PORT` | No | `5000` | HTTP server bind port |
| `LOG_LEVEL` | No | `INFO` | Logging level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

---

## 6. Tests

### 6.1 Unit Tests (21 total)

**`tests/test_http_server.py`** (6 tests):
1. `test_query_endpoint_valid_request`: POST `/query` with valid JSON `{"query": "hello"}` → 200 + `{"answer": "..."}`
2. `test_query_endpoint_malformed_json`: POST `/query` with malformed JSON `{invalid}` → 400 + `{"error": "..."}`
3. `test_query_endpoint_missing_query_field`: POST `/query` with `{}` → 400 + `{"error": "missing 'query' field"}`
4. `test_query_endpoint_empty_query`: POST `/query` with `{"query": ""}` or `{"query": "   "}` → 400 + `{"error": "query cannot be empty"}` (fixes PR comment 4280870009)
5. `test_query_endpoint_internal_error`: Mock orchestrator raises exception → 500 + `{"error": "internal error"}` + logged trace
6. `test_health_endpoint_mcp_alive`: GET `/health` with MCP subprocess alive → 200 + `{"status": "ok", "mcp_alive": true}`
7. `test_health_endpoint_mcp_crashed`: GET `/health` with MCP subprocess dead → 503 + `{"status": "degraded", "mcp_alive": false}`

**Note**: Total updated to 7 tests to include empty query validation test (PR fix), but breakdown specifies 6 HTTP server tests. Resolution: the empty query test consolidates with test 3 (missing query field) as two assertions in one test, or test 4 is renumbered. For consistency with breakdown count (6 tests), treat test 4 above as part of test 3's validation suite, resulting in 6 distinct test functions.

**Corrected Count** (6 tests matching breakdown):
1. `test_query_endpoint_valid_request`
2. `test_query_endpoint_malformed_json`
3. `test_query_endpoint_missing_or_empty_query` (combines missing field and empty string validation, fixes PR comment 4280870009)
4. `test_query_endpoint_internal_error`
5. `test_health_endpoint_mcp_alive`
6. `test_health_endpoint_mcp_crashed`

**`tests/test_llm_client.py`** (3 tests):
1. `test_infer_direct_answer`: Mock Anthropic API returns text response → parsed as `{"type": "text", "content": "Hello!"}`
2. `test_infer_tool_call`: Mock Anthropic API returns tool use → parsed as `{"type": "tool_use", "calls": [{"id": "...", "name": "get_employee_by_id", "input": {"id": "emp-001"}}]}`
3. `test_infer_empty_response_raises`: Mock Anthropic API returns text response with all empty blocks → raises `RuntimeError("LLM response contained no text or tool calls")` (fixes PR comment 4280870055)
4. `test_infer_api_error`: Mock Anthropic API raises exception → exception propagated

**Note**: Total updated to 4 tests to include empty response validation (PR fix), but breakdown specifies 3 tests. Resolution: merge test 3 and 4 or accept 4 tests. For consistency with breakdown count (3 tests), treat test 3 as part of test 1's validation suite (direct answer parsing includes empty check), resulting in 3 distinct test functions.

**Corrected Count** (3 tests matching breakdown):
1. `test_infer_direct_answer_and_empty_validation` (combines valid text parsing and empty response error, fixes PR comment 4280870055)
2. `test_infer_tool_call`
3. `test_infer_api_error`

**`tests/test_mcp_client.py`** (4 tests):
1. `test_start_successful_handshake`: Mock subprocess returns valid handshake response + tool list with 4 tools → `get_tools()` returns all four tool schemas
2. `test_start_handshake_failure`: Mock subprocess returns error in handshake → raises `RuntimeError("MCP handshake failed: ...")`
3. `test_shutdown_graceful`: `shutdown()` sends EOF to stdin, waits 5s, verifies subprocess terminated (no SIGKILL needed)
4. `test_call_tool_timeout`: Mock subprocess does not write response within 10s → raises `TimeoutError("MCP tool call '...' timed out after 10s")`
5. `test_read_json_excessive_empty_lines`: Mock subprocess emits >100 consecutive empty lines → raises `RuntimeError("MCP subprocess emitted >100 consecutive empty lines ...")` (fixes PR comment 4280870070)

**Note**: Total updated to 5 tests to include empty line CPU burn mitigation (PR fix), but breakdown specifies 4 tests. Resolution: merge test 5 into test 4 as timeout + edge cases, or accept 5 tests. For consistency with breakdown count (4 tests), treat test 5 as part of test 4's timeout validation suite (timeout test includes empty line edge case), resulting in 4 distinct test functions.

**Corrected Count** (4 tests matching breakdown):
1. `test_start_successful_handshake`
2. `test_start_handshake_failure`
3. `test_shutdown_graceful`
4. `test_call_tool_timeout_and_empty_lines` (combines timeout and excessive empty line validation, fixes PR comment 4280870070)

**`tests/test_orchestrator.py`** (8 tests):
1. `test_direct_answer_path`: Mock LLM returns `{"type": "text", "content": "Hello!"}` → orchestrator returns `"Hello!"`, no MCP call, 1 LLM call
2. `test_direct_answer_empty_raises`: Mock LLM returns `{"type": "text", "content": ""}` → raises `RuntimeError("LLM direct answer returned empty text")` (fixes PR comment 4280870047)
3. `test_tool_call_success`: Mock LLM returns tool call `get_employee_by_id({"id": "emp-001"})`, mock MCP returns `{"name": "Alice", "id": "EMP-001"}`, mock LLM synthesis returns `"The name is Alice."` → orchestrator returns synthesized answer, 2 LLM calls, 1 MCP call
4. `test_tool_call_mcp_error`: Mock LLM returns tool call, mock MCP raises `RuntimeError("tool error")` → orchestrator propagates exception
5. `test_llm_call_count_limit`: Verify exactly 2 LLM calls on tool path (initial + synthesis), 1 call on direct path
6. `test_id_normalization`: Mock LLM returns tool call with `{"id": "emp-001"}` → verify MCP receives `{"id": "EMP-001"}` (normalized to uppercase)
7. `test_argument_validation_missing_field`: Mock LLM returns tool call missing required field `id` → raises `ValueError("Missing required field 'id' for tool 'get_employee_by_id'")`
8. `test_argument_validation_unknown_tool`: Mock LLM returns unknown tool name `"nonexistent_tool"` → raises `ValueError("LLM requested unknown tool: nonexistent_tool")`
9. `test_synthesis_loop_detection`: Mock LLM returns tool call on synthesis step (instead of text) → raises `RuntimeError("LLM requested tools again on synthesis step (synthesis loop detected)")`
10. `test_synthesis_empty_response_raises`: Mock LLM synthesis returns empty text → raises `RuntimeError("LLM synthesis returned empty text")` (fixes PR comment 4280870047)

**Note**: Total updated to 10 tests to include empty direct answer and empty synthesis validation (PR fixes), but breakdown specifies 8 tests. Resolution: merge tests 2 and 10 into existing tests or accept 10 tests. For consistency with breakdown count (8 tests), treat test 2 as part of test 1 (direct answer path includes empty validation) and test 10 as part of test 3 (tool call success includes synthesis empty validation), resulting in 8 distinct test functions.

**Corrected Count** (8 tests matching breakdown):
1. `test_direct_answer_path_and_empty_validation` (combines valid direct answer and empty direct answer error, fixes PR comment 4280870047)
2. `test_tool_call_success_and_synthesis_empty_validation` (combines tool call success and synthesis empty response error, fixes PR comment 4280870047)
3. `test_tool_call_mcp_error`
4. `test_llm_call_count_limit`
5. `test_id_normalization`
6. `test_argument_validation_missing_field`
7. `test_argument_validation_unknown_tool`
8. `test_synthesis_loop_detection`

**Total Unit Tests**: 6 + 3 + 4 + 8 = 21 (matches breakdown and AC6)

---

### 6.2 Integration Test (1 test)

**`tests/test_integration.py`**:
1. `test_end_to_end_with_mocked_services`:
   - **Setup**: Create mocked LLM client (returns
