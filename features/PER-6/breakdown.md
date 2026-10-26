# Breakdown: MCP Server for GraphQL Data Fetching

**Epic:** [mcp] MCP server: tools that fetch data from the GraphQL service  
**Initiative:** Demo project  
**Date:** 2025-01-20

---

## Overview

This epic implements an MCP (Model Context Protocol) server that exposes four tools for querying a remote GraphQL service over HTTP. The server acts as a protocol adapter, translating MCP tool invocations into GraphQL queries and returning standardized JSON responses. All data originates from the GraphQL service; no local schemas or mock data are managed by this server.

The epic is implemented as **one story** (01-mcp) delivered through six incremental build steps that together satisfy all 14 acceptance criteria from requirements.md.

---

## Stories

### Story 01-mcp: Implement MCP Server with Four GraphQL Tools

**ID:** 1  
**Type:** story  
**Path:** `stories/01-mcp.md`  
**Repo:** mcp  
**After:** (none — first story in epic)

**Summary:** Implement the complete MCP server infrastructure, GraphQL client abstraction, and all four data-fetching tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) with comprehensive test coverage.

---

#### Acceptance Criteria

All 14 acceptance criteria from requirements.md apply to this story:

##### AC1: MCP tools are implemented

1. **Given** the MCP server is running  
   **When** an agent queries the available tools  
   **Then** the server exposes exactly four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`

2. **Given** the MCP server is running  
   **When** an agent invokes any of the four tools  
   **Then** the server accepts the request and attempts to execute it

##### AC2: Each tool queries the GraphQL server and returns a clear JSON result/error shape

3. **Given** `list_sales_order` is invoked  
   **When** the GraphQL server returns sales order data  
   **Then** the tool returns a JSON object with key `salesOrders` containing an array of sales orders

4. **Given** `list_employees` is invoked  
   **When** the GraphQL server returns employee data  
   **Then** the tool returns a JSON object with key `employees` containing an array of employees

5. **Given** `get_sales_order_by_id` is invoked with a valid ID  
   **When** the GraphQL server returns the matching sales order  
   **Then** the tool returns a JSON object with key `salesOrder` containing that sales order object

6. **Given** `get_sales_order_by_id` is invoked without an ID parameter  
   **When** the tool validates input  
   **Then** the tool returns `{"error": "Missing required parameter: id"}`

7. **Given** `get_sales_order_by_id` is invoked with an ID not found in the GraphQL service  
   **When** the GraphQL server returns no matching sales order  
   **Then** the tool returns `{"salesOrder": null}`

8. **Given** `get_employee_by_id` is invoked with a valid ID  
   **When** the GraphQL server returns the matching employee  
   **Then** the tool returns a JSON object with key `employee` containing that employee object

9. **Given** `get_employee_by_id` is invoked without an ID parameter  
   **When** the tool validates input  
   **Then** the tool returns `{"error": "Missing required parameter: id"}`

10. **Given** `get_employee_by_id` is invoked with an ID not found in the GraphQL service  
    **When** the GraphQL server returns no matching employee  
    **Then** the tool returns `{"employee": null}`

11. **Given** any tool is invoked  
    **When** the GraphQL server returns an error or is unreachable  
    **Then** the tool returns a JSON object with key `error` containing a clear error message

##### AC3: Automated tests with GraphQL client mocked or against local GraphQL

12. **Given** the test suite is executed  
    **When** tests run against a mocked GraphQL client  
    **Then** all four tools are verified to construct correct GraphQL queries and return expected JSON shapes for success cases

13. **Given** the test suite is executed  
    **When** tests simulate GraphQL errors (network failure, GraphQL error response, missing data)  
    **Then** all four tools are verified to return clear JSON error objects with key `error`

14. **Given** integration tests are executed  
    **When** tests run against a local GraphQL server  
    **Then** all four tools are verified to query the live GraphQL endpoint and return the expected JSON shapes

---

#### Allowed Paths

All paths relative to repo root: `/home/goos/projects/adlc-mcp-server`

**New source files:**

- `mcp/__init__.py` — Package marker
- `mcp/graphql_client.py` — GraphQL HTTP client abstraction
- `mcp/tools/__init__.py` — Package marker
- `mcp/tools/list_sales_order.py` — Tool handler for listing sales orders
- `mcp/tools/list_employees.py` — Tool handler for listing employees
- `mcp/tools/get_sales_order_by_id.py` — Tool handler for fetching sales order by ID
- `mcp/tools/get_employee_by_id.py` — Tool handler for fetching employee by ID
- `mcp/server.py` — MCP server entry point and tool registration

**New test files:**

- `tests/__init__.py` — Package marker
- `tests/test_graphql_client.py` — Unit tests for GraphQL client (4 tests)
- `tests/tools/__init__.py` — Package marker
- `tests/tools/test_list_sales_order.py` — Unit tests for list_sales_order tool (2 tests)
- `tests/tools/test_list_employees.py` — Unit tests for list_employees tool (2 tests)
- `tests/tools/test_get_sales_order_by_id.py` — Unit tests for get_sales_order_by_id tool (4 tests)
- `tests/tools/test_get_employee_by_id.py` — Unit tests for get_employee_by_id tool (4 tests)
- `tests/test_mcp_server.py` — Integration tests for MCP server (2 tests)

**Updated files:**

- `README.md` — Add setup instructions, usage examples, environment variable documentation

---

#### Verification Approach

**Unit tests (16 tests total):**

1. **GraphQL client tests (4 tests):**
   - Success response with valid data and no errors
   - HTTP error status code (4xx/5xx) returns error tuple
   - Network error (connection refused) raises NetworkError
   - Malformed JSON response raises ValueError

2. **Tool handler tests with mocked GraphQL client (12 tests):**
   - `list_sales_order` (2 tests):
     - Success with sales order data returns `{"salesOrders": [...]}`
     - GraphQL error response returns `{"error": "..."}`
   - `list_employees` (2 tests):
     - Success with employee data returns `{"employees": [...]}`
     - GraphQL error response returns `{"error": "..."}`
   - `get_sales_order_by_id` (4 tests):
     - Success with found sales order returns `{"salesOrder": {...}}`
     - Missing ID parameter returns `{"error": "Missing required parameter: id"}`
     - Not found (null data) returns `{"salesOrder": null}`
     - GraphQL error response returns `{"error": "..."}`
   - `get_employee_by_id` (4 tests):
     - Success with found employee returns `{"employee": {...}}`
     - Missing ID parameter returns `{"error": "Missing required parameter: id"}`
     - Not found (null data) returns `{"employee": null}`
     - GraphQL error response returns `{"error": "..."}`

**Integration tests (2 tests):**

3. **MCP server tests against local GraphQL service (2 tests):**
   - `list_sales_order` tool queries live GraphQL endpoint and returns valid JSON
   - `list_employees` tool queries live GraphQL endpoint and returns valid JSON

**Test execution:**

- Unit tests run with `pytest tests/ -k "not integration"` (GraphQL client mocked; no external dependencies)
- Integration tests run with `pytest tests/test_mcp_server.py` (requires `GRAPHQL_ENDPOINT` environment variable set to local GraphQL service)

**Expected test count:** 18 automated tests total (4 GraphQL client + 12 tool handlers + 2 integration)

---

#### Dependencies

**Requires:**

- GraphQL service (HTTP endpoint) provided by PER-5 / adlc-graphql must be running for integration tests
- Python `requests` library for HTTP communication
- MCP SDK for Python for protocol implementation
- `pytest` for test execution
- `unittest.mock` for mocking GraphQL client in unit tests

**Blocks:**

- None (single-story epic)

---

#### Notes

- The GraphQL service endpoint is configured via `GRAPHQL_ENDPOINT` environment variable; the server fails fast at startup if this is not set
- All tool output shapes are JSON-serializable dicts (no custom classes exposed to MCP runtime)
- Error responses follow consistent shape: `{"error": "<message>"}` for all failure modes
- GraphQL queries are constructed as strings within tool handlers (no external query files)
- The story is delivered through six incremental build steps as described in the story LLD
- No authentication or authorization implemented (out of scope per ticket)
- The GraphQL service schema is owned by PER-5; this server consumes it as a black box over HTTP

---

## Epic Completion Criteria

All 14 acceptance criteria from requirements.md are satisfied by story 01-mcp. The epic is considered complete when:

1. All 18 automated tests pass
2. The MCP server successfully registers and serves all four tools
3. Integration tests verify end-to-end communication with the local GraphQL service
4. README.md documents setup, configuration, and usage

---

## Out of Scope

- Authentication mechanisms
- Re-implementing GraphQL schemas or mock data (covered by PER-5 / adlc-graphql)
- Agent orchestration UI or runtime beyond tool invocation
- Additional tools beyond the four specified in the ticket

---

## Open Questions

None.
