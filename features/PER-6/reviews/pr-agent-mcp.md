```json
[
  {
    "path": "mcp/server.py",
    "line": 23,
    "body": "**Module shadowing hazard:** The `_load_sdk()` function manipulates `sys.modules` and `sys.path` to avoid shadowing the official MCP SDK with the local `mcp` package. This complex workaround is fragile and untested. The requirements and HLD do not specify or justify this approach.\n\n**Risk:** The module-swapping logic may fail in edge cases (e.g., concurrent imports, cached modules, different Python versions), causing import errors or silent failures at runtime.\n\n**AC gap:** AC1 requires that the server \"accepts the request and attempts to execute it\" when tools are invoked. If SDK loading fails, the server will not start, violating this acceptance criterion.\n\n**Recommendation:** Rename the local `mcp` package to `mcp_server` or `graphql_mcp` to eliminate the naming conflict entirely. This is the standard Python practice and removes the need for runtime module manipulation."
  },
  {
    "path": "tests/test_mcp_server.py",
    "line": 1,
    "body": "**Missing integration test coverage:** AC3 requires 18 automated tests total: 4 GraphQL client unit tests + 12 tool unit tests + **2 integration tests**. The requirements specify that integration tests \"verify all four tools\" against a live GraphQL endpoint.\n\n**Current state:** Only 2 integration tests exist (`test_list_sales_order_integration` and `test_list_employees_integration`), covering 2 of 4 tools.\n\n**AC gap:** The `get_sales_order_by_id` and `get_employee_by_id` tools are not verified in integration tests. AC3 states: \"When tests run against a local GraphQL server, **all four tools** are verified to query the live GraphQL endpoint and return the expected JSON shapes.\"\n\n**Missing tests:**\n- `test_get_sales_order_by_id_integration()` — verify tool queries live endpoint with a valid ID and returns `{\"salesOrder\": {...}}`\n- `test_get_employee_by_id_integration()` — verify tool queries live endpoint with a valid ID and returns `{\"employee\": {...}}`"
  },
  {
    "path": "tests/tools/test_get_sales_order_by_id.py",
    "line": 40,
    "body": "**Test gap:** The unit test `test_get_sales_order_by_id_missing_id` checks `None` and whitespace-only strings (`\"  \"`), but the requirements explicitly state the error case is \"invoked **without an ID parameter**\" (AC2, criterion 6).\n\n**Issue:** The test does not verify the behavior when the `id` key is absent from the arguments dict entirely (e.g., `get_sales_order_by_id(client, arguments={})` in `server.py:123` where `arguments.get(\"id\")` returns `None`).\n\n**Current test:** Only checks `get_sales_order_by_id(client, None)` directly. Does not verify the contract between `server.py` and the tool handler.\n\n**Recommendation:** Add a test case that calls `get_sales_order_by_id(client, \"\")` or documents that `None` from `dict.get()` is the canonical \"missing parameter\" case. The current test coverage is sufficient if `arguments.get(\"id\")` always passes `None` when the key is absent, but this is not explicitly verified."
  },
  {
    "path": "tests/tools/test_get_employee_by_id.py",
    "line": 38,
    "body": "**Test gap:** Same issue as `test_get_sales_order_by_id.py`. The unit test `test_get_employee_by_id_missing_id` checks `None` and empty strings, but does not verify the contract between `server.py:124` (`arguments.get(\"id\")`) and the tool handler.\n\n**AC reference:** AC2, criterion 9: \"Given `get_employee_by_id` is invoked **without an ID parameter**, When the tool validates input, Then the tool returns `{\"error\": \"Missing required parameter: id\"}`.\"\n\n**Recommendation:** Same as above — verify that `arguments.get(\"id\")` passing `None` (when key is absent) is handled correctly, or add a test that explicitly checks this case."
  },
  {
    "path": "mcp/graphql_client.py",
    "line": 76,
    "body": "**Spec alignment:** The HLD specifies that when the GraphQL service returns null data for a single-resource query, the tool should return `{\"<resource>\": null}` (AD-4: Null Data vs. Error Distinction). However, the GraphQL client returns `(data, errors)` where `data` can be `None` or a dict containing null fields.\n\n**Issue:** The client does not distinguish between:\n1. `{\"data\": null, \"errors\": [...]}` → `data = None` (query failed)\n2. `{\"data\": {\"salesOrder\": null}}` → `data = {\"salesOrder\": null}` (resource not found)\n\nBoth cases are valid GraphQL responses per the spec, but the tool handlers treat `data is None` as an error (see `get_sales_order_by_id.py:46` and `get_employee_by_id.py:46`).\n\n**AC impact:** AC2, criterion 7 and 10 require that when a resource is \"not found\", the tool returns `{\"<resource>\": null}`. If the GraphQL service returns `{\"data\": null}` (a valid response per GraphQL spec when a resolver returns null), the tools will return `{\"error\": \"no data returned\"}` instead of `{\"<resource>\": null}`.\n\n**Clarification needed:** Does the GraphQL service from PER-5 return `{\"data\": null}` or `{\"data\": {\"salesOrder\": null}}` when a resource is not found? The current implementation assumes the latter. If PER-5 returns the former, the tools will fail AC2 criteria 7 and 10."
  },
  {
    "path": "requirements.txt",
    "line": 2,
    "body": "**Dependency pinning:** The `mcp` SDK is pinned to `>=1.0,<2`, but the HLD states \"MCP protocol compliance is assumed; refer to MCP specification for tool registration and invocation contracts.\" The requirements do not specify which MCP SDK version is required.\n\n**Risk:** MCP SDK 1.x may introduce breaking changes in minor versions. The code uses `mcp.server.Server`, `mcp.types`, and `mcp.server.stdio.stdio_server`, but does not document which MCP SDK version was tested.\n\n**Recommendation:** Pin to a specific minor version (e.g., `mcp>=1.0,<1.1`) or document the tested SDK version in README.md. The current range allows any 1.x version, which may break compatibility."
  }
]
```
