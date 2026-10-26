Fix these forge PR review comments (auto-route):

### agent comment 4280870009
**Malformed JSON validation incomplete**: The handler checks `request.get_json(silent=True)` for `None` but does not verify that `payload.get("query")` is non-empty. AC states the endpoint should return 400 for missing `query` field; an empty string `{"query": ""}` will pass validation and reach the orchestrator, which may produce a degenerate answer rather than rejecting the request. Add: `or not payload["query"].strip()` after the `isinstance` check on line 30.

### agent comment 4280870021
**Empty environment variable fallback inconsistent**: `base_url` defaults to `"https://api.anthropic.com"` when `ANTHROPIC_BASE_URL` is unset or empty (line 62), but `model` defaults to `"claude-3-5-sonnet-20241022"` only when unset (line 65) — an explicitly empty `ANTHROPIC_MODEL` will produce an empty string, causing an Anthropic API error. The story's env var table marks `ANTHROPIC_MODEL` as optional with a default; the code should treat empty the same as unset. Change line 65 to match line 62's pattern: `model = os.environ.get("ANTHROPIC_MODEL", "claude-3-5-sonnet-20241022").strip() or "claude-3-5-sonnet-20241022"`.

### agent comment 4280870047
**Synthesis response validation assumes single content block**: Line 76 checks `synthesis_response["type"] != "text"` but does not verify that `synthesis_response["content"]` is non-empty. The LLM client's `_parse_response` can return `{"type": "text", "content": ""}` if the API sends an empty text block. The orchestrator will return an empty string as the final answer, which violates AC1's implicit contract that the answer is meaningful text. Add: `if not synthesis_response.get("content"):` before line 77 to raise `RuntimeError("LLM synthesis returned empty text")`.

### agent comment 4280870055
**Empty text concatenation returns empty string**: Lines 116–122 join text blocks and return the result, but if all text blocks are empty strings, the function returns `{"type": "text", "content": ""}` rather than raising an error. The orchestrator will then return an empty answer to the user. AC2 implies a direct answer is "text" (non-empty), but the code does not enforce it. Either raise `RuntimeError("LLM response contained no text or tool calls")` after line 122 when `texts` is empty, or let the orchestrator validate non-empty content (see finding 4).

### agent comment 4280870070
**Tight timeout loop burns CPU**: The `_read_json` loop (lines 201–225) uses `select.select` with a recalculated timeout on each iteration, but if `stdout.readline()` returns an empty line (e.g. the MCP server emits blank lines as keepalives), the loop re-enters `select` immediately with the remaining timeout. If the server emits many blank lines near the deadline, the loop may spin in a tight CPU burn checking `time.monotonic()` on every iteration. Mitigation: track the number of consecutive empty lines and raise `RuntimeError` after a threshold (e.g. 100 empty lines) to prevent infinite spin on a misbehaving server.

### agent comment 4280870077
**Missing test: empty `query` string**: AC states the endpoint should validate the `query` field is present, but the test suite does not cover `{"query": ""}` (empty string). The current code (finding 1) will accept it. Add a test: `client.post("/query", json={"query": ""})` expecting 400.

### agent comment 4280870086
**ID normalization test does not verify lowercase input**: `test_id_normalization` asserts that `mcp.call_tool` receives `{"id": "EMP-001"}`, but the mock LLM response `TOOL_CALL` (line 18) already contains `"id": "emp-001"` lowercase. The test verifies the normalization happened, but does not assert that the orchestrator received lowercase input and normalized it. A refactor that skips normalization when the input is already uppercase would still pass. Assert that `llm.infer.call_args_list[0]` shows the original query contained lowercase `emp-001`.

### agent comment 4280870095
**Missing test: tool call success path**: The MCP client test suite covers subprocess lifecycle (start, shutdown, timeout) but does not include a test for a successful `call_tool` invocation with a real subprocess that returns a valid result. The `test_call_tool_timeout` test patches `select.select` to never return ready, preventing any actual tool call. Add a test: spawn the success script, call `client.call_tool("get_employee_by_id", {"id": "EMP-001"})`, and verify the script receives the JSON-RPC request and the client returns the parsed result. Without this, a bug in the request/response serialization could go undetected.

### agent comment 4280870102
**Inconsistent environment variable documentation**: The README table (lines 33–43) lists `ANTHROPIC_MODEL` with default `claude-3-5-sonnet-20241022`, but the code (finding 2) does not treat an explicitly empty `ANTHROPIC_MODEL` the same as unset. The README should note: "Empty values for optional variables fall back to the default." This aligns user expectations with the intended behavior from the story's env var table.
