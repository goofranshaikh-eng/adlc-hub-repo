Fix these forge PR review comments (auto-route):

### mcp comment 4278914730
**AC violation: Missing InitializationOptions usage.** AC1 requires the server to expose exactly four tools and accept invocations. The LLD (Story 01-mcp, Build Step 6) specifies:

> "Create InitializationOptions object with server name and version metadata … Pass init options to server.run() as documented in MCP SDK."

The code creates `init_options` via `server.create_initialization_options()` but then only asserts its type without using it. Line 141 checks:

```python
assert isinstance(init_options, InitializationOptions) or init_options is not None
```

**Expected:** Pass `init_options` to `server.run()` per the SDK contract. The assertion does nothing except reference the import to satisfy a static check.

**Impact:** The server may fail to initialize properly with MCP clients expecting metadata, breaking AC1's "server is running" precondition.

**Fix:** Remove the assertion and ensure `init_options` is passed correctly to `server.run()` (line 143 already does this, so just remove line 141).

### mcp comment 4278914741
**AC2 error shape inconsistency.** AC2 requires:

> "the tool returns a JSON object with key `error` containing a clear error message"

The HLD Error Handling section specifies:

> "HTTP errors (4xx, 5xx status codes) → `{"error": "Failed to fetch <resource>: HTTP error <status>"}`"

The GraphQL client returns `[{"message": "HTTP error: {status}"}]` in the errors list, but tool handlers expect to extract `errors[0].get("message")` and format it as `"Failed to fetch <resource>: <message>"`. This works for GraphQL errors but produces inconsistent error messages for HTTP errors.

**Current output for HTTP 500:**
```python
{"error": "Failed to fetch sales orders: HTTP error: 500"}
```

**Expected per HLD:**
```python
{"error": "Failed to fetch sales orders: HTTP error 500"}
```

The extra colon after "HTTP error" breaks the documented error message format.

**Fix:** Line 66 should return `[{"message": f"HTTP error {response.status_code}"}]` (no colon after "error").

### mcp comment 4278914747
**Missing AC3 test coverage: malformed response shape.** AC3 requires:

> "tests simulate GraphQL errors (network failure, GraphQL error response, missing data)"

The HLD Error Handling section lists six error cases, including:

> "Malformed responses (invalid JSON) → `{"error": "Invalid response from GraphQL service: <message>"}`"

The test suite has four GraphQL client tests:
1. Success response ✓
2. HTTP error ✓
3. Network error ✓
4. Malformed JSON (tests `ValueError` is raised) ✓

But it's missing a test for the case where JSON decodes successfully but is not a dict (e.g., the response is `[]` or `"string"`). Line 72–73 of `graphql_client.py` handle this:

```python
if not isinstance(body, dict):
    raise ValueError("malformed JSON response: expected object")
```

**Missing test case:**

```python
def test_execute_query_non_dict_json():
    client = GraphQLClient("http://example.com/graphql")
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = []  # Valid JSON, wrong type
    
    with patch("mcp.graphql_client.requests.post", return_value=mock_response):
        with pytest.raises(ValueError, match="expected object"):
            client.execute_query("{ salesOrders { id } }")
```

**Impact:** The error path at line 72–73 is untested, violating AC3's requirement to verify all error cases.
