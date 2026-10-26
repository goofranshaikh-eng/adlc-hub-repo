```json
[
  {
    "path": "app/agent/http_server.py",
    "line": 29,
    "body": "**AC gap: empty query validation not required by AC1.** AC1 states the endpoint accepts `{\"query\": \"<natural language>\"}` but does not mandate rejection of empty strings. The implementation adds a stricter validation (`if not payload[\"query\"].strip()`) beyond the spec. While defensible, this is not an AC requirement and could reject legitimate edge cases (e.g. whitespace-only queries that the LLM might handle). Mark this as a clarification or remove the check unless the team confirms it as intended behavior."
  },
  {
    "path": "app/agent/http_server.py",
    "line": 64,
    "body": "**Environment variable handling diverges from requirements contract.** Requirements.md specifies `ANTHROPIC_BASE_URL` is optional with default `https://api.anthropic.com`, and empty is treated as unset. The implementation uses `.strip() or \"https://api.anthropic.com\"` which correctly handles empty strings, but this logic is duplicated for `ANTHROPIC_MODEL` (lines 68–70) which is **not listed in requirements.md** at all. The code introduces a new env var without spec approval. If `ANTHROPIC_MODEL` is intentional, requirements.md must be updated; otherwise remove it and hardcode the model per LLMClient's `DEFAULT_MODEL`."
  },
  {
    "path": "app/agent/llm_client.py",
    "line": 12,
    "body": "**Model configurability not in requirements.** `LLMClient.__init__` accepts a `model` parameter and uses it in `infer()`, but requirements.md does not list `ANTHROPIC_MODEL` as a supported environment variable. The README.md (modified in this PR) adds `ANTHROPIC_MODEL` to the env var table, but the requirements artifact was not updated. Either remove the `model` parameter and hardcode `DEFAULT_MODEL`, or update requirements.md to document `ANTHROPIC_MODEL` as an optional env var."
  },
  {
    "path": "app/agent/mcp_client.py",
    "line": 215,
    "body": "**Infinite-loop guard threshold (100 empty lines) is arbitrary and undocumented.** The code raises `RuntimeError` after 101 consecutive empty lines to prevent a \"possible keepalive loop,\" but this threshold is not derived from any MCP spec requirement or requirements.md AC. The comment references a \"keepalive loop\" scenario that is not described anywhere in the epic. This feels like defensive coding without a concrete failure mode. Either document the failure case in HLD or remove the guard and rely on the existing timeout mechanism."
  },
  {
    "path": "tests/test_http_server.py",
    "line": 34,
    "body": "**Test covers empty-query validation not required by AC.** AC1 and AC6 do not mandate rejection of empty queries. This test enforces stricter behavior than the spec (`test_query_endpoint_missing_or_empty_query` covers `\"\"` and `\"   \"`). If the team decides empty queries are invalid, update requirements.md AC1 or add a clarification note; otherwise this test is testing implementation details, not acceptance criteria."
  },
  {
    "path": "tests/test_orchestrator.py",
    "line": 37,
    "body": "**Test covers empty-answer validation beyond AC scope.** AC2/AC3 require the agent to return answers but do not explicitly require rejection of empty LLM responses. The tests `test_direct_answer_path_and_empty_validation` and `test_tool_call_success_and_synthesis_empty_validation` enforce that empty or whitespace-only answers raise `RuntimeError`. While this is reasonable defensive behavior, it is not an AC requirement. Mark these cases as clarifications in requirements.md or document the rationale in HLD §7 (risk mitigation)."
  },
  {
    "path": "app/agent/orchestrator.py",
    "line": 60,
    "body": "**ID normalization is hardcoded to `id` field name only.** AC5 states \"the agent normalizes the ID before calling the tool,\" and the implementation applies `.upper()` to any argument key named `id`. However, the MCP tools may use other field names for identifiers (e.g. `employee_id`, `order_id`). The current logic is fragile: it only works because the GraphQL mock happens to use `id` everywhere. If PER-6 tools later introduce `employee_id` or `order_id` fields, this normalization will silently fail. Either: (1) make the field name configurable per tool schema, or (2) clarify in HLD AD-3 that only fields literally named `id` are normalized, and update AC5 to reflect this constraint."
  },
  {
    "path": "README.md",
    "line": 37,
    "body": "**README documents `ANTHROPIC_MODEL` env var not in requirements.** The table lists `ANTHROPIC_MODEL` as optional with default `claude-3-5-sonnet-20241022`, but requirements.md does not mention this variable. Either add it to requirements.md or remove it from README and hardcode the model in `LLMClient`."
  },
  {
    "path": "tests/manual_verification.md",
    "line": 1,
    "body": "**Manual verification is not a test artifact per AC6 contract.** AC6 states \"a documented manual verification script successfully demonstrates AC1–AC5 against real PER-5/PER-6 services.\" The word \"successfully\" implies the script must have been *run* and *passed*, not just documented. This PR adds the documentation but does not show evidence of execution (e.g. a timestamped log or screenshot). The QA report should confirm the script was run before merge, or AC6 is not met."
  },
  {
    "path": "app/agent/http_server.py",
    "line": 75,
    "body": "**MCP server command default logic duplicates the constant.** Line 75 uses `os.environ.get(\"MCP_SERVER_COMMAND\", DEFAULT_MCP_COMMAND).strip() or DEFAULT_MCP_COMMAND`, but `DEFAULT_MCP_COMMAND` is already defined as `\"python -m mcp.server\"` on line 15. The `.strip() or DEFAULT_MCP_COMMAND` pattern handles empty strings, which is consistent with other env vars, but the redundancy could be simplified to a single `.strip() or \"python -m mcp.server\"` inline. Minor clarity issue, not a functional bug."
  }
]
```
