# Story 01-mcp: Implement MCP Server with Four GraphQL Tools

**Repo:** mcp  
**Epic:** [mcp] MCP server: tools that fetch data from the GraphQL service  
**Path:** `stories/01-mcp.md`

---

## Acceptance Criteria

All acceptance criteria from requirements.md apply to this story:

### AC1: MCP tools are implemented

1. **Given** the MCP server is running  
   **When** an agent queries the available tools  
   **Then** the server exposes exactly four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`

2. **Given** the MCP server is running  
   **When** an agent invokes any of the four tools  
   **Then** the server accepts the request and attempts to execute it

### AC2: Each tool queries the GraphQL server and returns a clear JSON result/error shape

3. **Given** `list_sales_order` is invoked  
   **When** the GraphQL server returns sales order data  
   **Then** the tool returns `{"salesOrders": [...]}`

4. **Given** `list_employees` is invoked  
   **When** the GraphQL server returns employee data  
   **Then** the tool returns `{"employees": [...]}`

5. **Given** `get_sales_order_by_id` is invoked with a valid ID  
   **When** the GraphQL server returns the matching sales order  
   **Then** the tool returns `{"salesOrder": {...}}`

6. **Given** `get_sales_order_by_id` is invoked without an ID parameter  
   **When** the tool validates input  
   **Then** the tool returns `{"error": "Missing required parameter: id"}`

7. **Given** `get_sales_order_by_id` is invoked with an ID not found in the GraphQL service  
   **When** the GraphQL server returns no matching sales order  
   **Then** the tool returns `{"salesOrder": null}`

8. **Given** `get_employee_by_id` is invoked with a valid ID  
   **When** the GraphQL server returns the matching employee  
   **Then** the tool returns `{"employee": {...}}`

9. **Given** `get_employee_by_id` is invoked without an ID parameter  
   **When** the tool validates input  
   **Then** the tool returns `{"error": "Missing required parameter: id"}`

10. **Given** `get_employee_by_id` is invoked with an ID not found in the GraphQL service  
    **When** the GraphQL server returns no matching employee  
    **Then** the tool returns `{"employee": null}`

11. **Given** any tool is invoked  
    **When** the GraphQL server returns an error or is unreachable  
    **Then** the tool returns `{"error": "<message>"}`

### AC3: Automated tests with GraphQL client mocked or against local GraphQL

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

## LLD

### Overview

This story implements the complete MCP server for GraphQL data fetching. The implementation provides four MCP tools that query a remote GraphQL service over HTTP and return standardized JSON responses. The story is delivered through six incremental build steps that together satisfy all 14 acceptance criteria.

The MCP server acts as a protocol adapter between the MCP protocol (tool invocations) and the GraphQL service (HTTP queries), translating tool calls into GraphQL queries and normalizing responses into consistent JSON shapes.

### File Plan

All paths relative to repo root: `/home/goos/projects/adlc-mcp-server`

#### New files (source)

- `mcp/__init__.py` — Package marker (empty)
- `mcp/graphql_client.py` — GraphQL HTTP client abstraction
- `mcp/tools/__init__.py` — Package marker (empty)
- `mcp/tools/list_sales_order.py` — Tool handler for listing sales orders
- `mcp/tools/list_employees.py` — Tool handler for listing employees
- `mcp/tools/get_sales_order_by_id.py` — Tool handler for fetching sales order by ID
- `mcp/tools/get_employee_by_id.py` — Tool handler for fetching employee by ID
- `mcp/server.py` — MCP server entry point and tool registration

#### New files (tests)

- `tests/__init__.py` — Package marker (empty)
- `tests/test_graphql_client.py` — Unit tests for GraphQL client (4 tests)
- `tests/tools/__init__.py` — Package marker (empty)
- `tests/tools/test_list_sales_order.py` — Unit tests for list_sales_order tool (2 tests)
- `tests/tools/test_list_employees.py` — Unit tests for list_employees tool (2 tests)
- `tests/tools/test_get_sales_order_by_id.py` — Unit tests for get_sales_order_by_id tool (4 tests)
- `tests/tools/test_get_employee_by_id.py` — Unit tests for get_employee_by_id tool (4 tests)
- `tests/test_mcp_server.py` — Integration tests for MCP server (2 tests)

#### Updated files

- `README.md` — Add setup instructions, usage examples, environment variable documentation

---

### Component Interfaces

#### GraphQL Client (`mcp/graphql_client.py`)

**Purpose:** Abstract HTTP communication with the GraphQL service. Returns parsed JSON responses.

**Interface:**

```python
class NetworkError(Exception):
    """Raised when network communication fails."""
    pass

class GraphQLClient:
    def __init__(self, endpoint: str):
        """
        Initialize client with GraphQL endpoint URL.
        
        Args:
            endpoint: GraphQL service HTTP endpoint URL
            
        Raises:
            ValueError: if endpoint is empty or None
        """
        
    def execute_query(self, query: str, variables: dict = None) -> tuple:
        """
        Send GraphQL query via HTTP POST.
        
        Args:
            query: GraphQL query string
            variables: Optional query variables dict
            
        Returns:
            Tuple (data, errors) where:
            - data: Response data dict or None
            - errors: List of error dicts or empty list
            
        Raises:
            NetworkError: if HTTP request fails (connection refused, timeout)
            ValueError: if response JSON is malformed
        """
```

**Configuration:**

- Reads `GRAPHQL_ENDPOINT` environment variable for endpoint URL
- Uses Python `requests` library for HTTP POST
- Content-Type: `application/json`
- Parses response JSON: extracts `data` and `errors` fields per GraphQL spec

**Error handling:**

- Network errors (connection refused, timeout, DNS failure) → raise `NetworkError` with clear message
- HTTP error status codes (4xx, 5xx) → return `(None, [{"message": "HTTP error: <status_code>"}])`
- Malformed JSON response → raise `ValueError` with message

**Dependencies:**

- `requests` library for HTTP communication
- `json` module for payload serialization

---

#### Tool Handler: `list_sales_order`

**Module:** `mcp/tools/list_sales_order.py`

**Function signature:**

```python
def list_sales_order(graphql_client: GraphQLClient) -> dict:
    """
    Query GraphQL service for all sales orders.
    
    Args:
        graphql_client: Initialized GraphQLClient instance
        
    Returns:
        Success: {"salesOrders": [...]}
        Error: {"error": "<message>"}
    """
```

**GraphQL query:**

```graphql
query {
  salesOrders {
    id
    customerId
    total
    status
  }
}
```

**Output shapes:**

- Success: `{"salesOrders": [{"id": "SO-001", "customerId": "C-123", "total": 1500.0, "status": "shipped"}, ...]}`
- Empty result: `{"salesOrders": []}`
- Error: `{"error": "Failed to fetch sales orders: <reason>"}`

**Algorithm:**

1. Construct GraphQL query string (no variables)
2. Call `graphql_client.execute_query(query)`
3. Catch `NetworkError` → return `{"error": "GraphQL service unavailable: <message>"}`
4. Catch `ValueError` → return `{"error": "Invalid response from GraphQL service: <message>"}`
5. If `errors` list is non-empty → return `{"error": "Failed to fetch sales orders: <first_error_message>"}`
6. If `data` is None or `data["salesOrders"]` is None → return `{"error": "Failed to fetch sales orders: no data returned"}`
7. Return `{"salesOrders": data["salesOrders"]}`

---

#### Tool Handler: `list_employees`

**Module:** `mcp/tools/list_employees.py`

**Function signature:**

```python
def list_employees(graphql_client: GraphQLClient) -> dict:
    """
    Query GraphQL service for all employees.
    
    Args:
        graphql_client: Initialized GraphQLClient instance
        
    Returns:
        Success: {"employees": [...]}
        Error: {"error": "<message>"}
    """
```

**GraphQL query:**

```graphql
query {
  employees {
    id
    name
    department
    email
  }
}
```

**Output shapes:**

- Success: `{"employees": [{"id": "E-001", "name": "Alice Smith", "department": "Engineering", "email": "alice@example.com"}, ...]}`
- Empty result: `{"employees": []}`
- Error: `{"error": "Failed to fetch employees: <reason>"}`

**Algorithm:**

1. Construct GraphQL query string (no variables)
2. Call `graphql_client.execute_query(query)`
3. Catch `NetworkError` → return `{"error": "GraphQL service unavailable: <message>"}`
4. Catch `ValueError` → return `{"error": "Invalid response from GraphQL service: <message>"}`
5. If `errors` list is non-empty → return `{"error": "Failed to fetch employees: <first_error_message>"}`
6. If `data` is None or `data["employees"]` is None → return `{"error": "Failed to fetch employees: no data returned"}`
7. Return `{"employees": data["employees"]}`

---

#### Tool Handler: `get_sales_order_by_id`

**Module:** `mcp/tools/get_sales_order_by_id.py`

**Function signature:**

```python
def get_sales_order_by_id(graphql_client: GraphQLClient, id: str) -> dict:
    """
    Query GraphQL service for a sales order by ID.
    
    Args:
        graphql_client: Initialized GraphQLClient instance
        id: Sales order ID (required)
        
    Returns:
        Success (found): {"salesOrder": {...}}
        Success (not found): {"salesOrder": null}
        Error: {"error": "<message>"}
    """
```

**GraphQL query:**

```graphql
query GetSalesOrder($id: ID!) {
  salesOrder(id: $id) {
    id
    customerId
    total
    status
  }
}
```

**Input validation:**

- If `id` is None or empty string → return `{"error": "Missing required parameter: id"}`

**Output shapes:**

- Found: `{"salesOrder": {"id": "SO-001", "customerId": "C-123", "total": 1500.0, "status": "shipped"}}`
- Not found: `{"salesOrder": null}`
- Missing ID: `{"error": "Missing required parameter: id"}`
- Error: `{"error": "Failed to fetch sales order <id>: <reason>"}`

**Algorithm:**

1. Validate `id` parameter:
   - If `id` is None or `id.strip() == ""` → return `{"error": "Missing required parameter: id"}`
2. Construct GraphQL query string with variable placeholder
3. Construct variables dict: `{"id": id}`
4. Call `graphql_client.execute_query(query, variables)`
5. Catch `NetworkError` → return `{"error": "GraphQL service unavailable: <message>"}`
6. Catch `ValueError` → return `{"error": "Invalid response from GraphQL service: <message>"}`
7. If `errors` list is non-empty → return `{"error": "Failed to fetch sales order <id>: <first_error_message>"}`
8. If `data` is None → return `{"error": "Failed to fetch sales order <id>: no data returned"}`
9. Return `{"salesOrder": data["salesOrder"]}` (note: `data["salesOrder"]` may be `null`)

---

#### Tool Handler: `get_employee_by_id`

**Module:** `mcp/tools/get_employee_by_id.py`

**Function signature:**

```python
def get_employee_by_id(graphql_client: GraphQLClient, id: str) -> dict:
    """
    Query GraphQL service for an employee by ID.
    
    Args:
        graphql_client: Initialized GraphQLClient instance
        id: Employee ID (required)
        
    Returns:
        Success (found): {"employee": {...}}
        Success (not found): {"employee": null}
        Error: {"error": "<message>"}
    """
```

**GraphQL query:**

```graphql
query GetEmployee($id: ID!) {
  employee(id: $id) {
    id
    name
    department
    email
  }
}
```

**Input validation:**

- If `id` is None or empty string → return `{"error": "Missing required parameter: id"}`

**Output shapes:**

- Found: `{"employee": {"id": "E-001", "name": "Alice Smith", "department": "Engineering", "email": "alice@example.com"}}`
- Not found: `{"employee": null}`
- Missing ID: `{"error": "Missing required parameter: id"}`
- Error: `{"error": "Failed to fetch employee <id>: <reason>"}`

**Algorithm:**

1. Validate `id` parameter:
   - If `id` is None or `id.strip() == ""` → return `{"error": "Missing required parameter: id"}`
2. Construct GraphQL query string with variable placeholder
3. Construct variables dict: `{"id": id}`
4. Call `graphql_client.execute_query(query, variables)`
5. Catch `NetworkError` → return `{"error": "GraphQL service unavailable: <message>"}`
6. Catch `ValueError` → return `{"error": "Invalid response from GraphQL service: <message>"}`
7. If `errors` list is non-empty → return `{"error": "Failed to fetch employee <id>: <first_error_message>"}`
8. If `data` is None → return `{"error": "Failed to fetch employee <id>: no data returned"}`
9. Return `{"employee": data["employee"]}` (note: `data["employee"]` may be `null`)

---

#### MCP Server (`mcp/server.py`)

**Purpose:** Register and serve all four tools via MCP protocol.

**Initialization algorithm:**

1. Read `GRAPHQL_ENDPOINT` environment variable
   - If not set or empty → log error "GRAPHQL_ENDPOINT environment variable not set" and exit with status 1
2. Instantiate `GraphQLClient` with endpoint URL
   - Catch `ValueError` (invalid endpoint) → log error "Invalid GraphQL endpoint: <message>" and exit with status 1
3. Initialize MCP server runtime (protocol-specific setup using MCP SDK)
4. Register four tool handlers with MCP server:
   - `list_sales_order` (no input parameters)
   - `list_employees` (no input parameters)
   - `get_sales_order_by_id` (requires `id` string parameter)
   - `get_employee_by_id` (requires `id` string parameter)
5. Start MCP server (listen for tool invocation requests on standard I/O or configured transport)

**Tool registration schema (MCP protocol):**

- `list_sales_order`:
  - Input schema: none (empty object)
  - Handler: calls `list_sales_order(graphql_client)` and returns result
- `list_employees`:
  - Input schema: none (empty object)
  - Handler: calls `list_employees(graphql_client)` and returns result
- `get_sales_order_by_id`:
  - Input schema: `{"type": "object", "properties": {"id": {"type": "string"}}, "required": ["id"]}`
  - Handler: extracts `id` from input dict, calls `get_sales_order_by_id(graphql_client, input["id"])`, returns result
- `get_employee_by_id`:
  - Input schema: `{"type": "object", "properties": {"id": {"type": "string"}}, "required": ["id"]}`
  - Handler: extracts `id` from input dict, calls `get_employee_by_id(graphql_client, input["id"])`, returns result

**Main entry point:**

```python
def main():
    """
    MCP server entry point.
    
    Reads GRAPHQL_ENDPOINT from environment, initializes GraphQL client,
    registers all four tools, and starts the MCP server.
    
    Exits with status 1 if GRAPHQL_ENDPOINT is not set or client
    initialization fails.
    """
    # 1. Read GRAPHQL_ENDPOINT
    endpoint = os.getenv("GRAPHQL_ENDPOINT")
    if not endpoint:
        logging.error("GRAPHQL_ENDPOINT environment variable not set")
        sys.exit(1)
    
    # 2. Initialize GraphQL client
    try:
        graphql_client = GraphQLClient(endpoint)
    except ValueError as e:
        logging.error(f"Invalid GraphQL endpoint: {e}")
        sys.exit(1)
    
    # 3. Initialize MCP server runtime
    server = Server("mcp-graphql-server")
    
    # 4. Register tool handlers
    @server.list_tools()
    async def handle_list_tools() -> list[types.Tool]:
        return [
            types.Tool(
                name="list_sales_order",
                description="Query GraphQL service for all sales orders",
                inputSchema={"type": "object", "properties": {}}
            ),
            types.Tool(
                name="list_employees",
                description="Query GraphQL service for all employees",
                inputSchema={"type": "object", "properties": {}}
            ),
            types.Tool(
                name="get_sales_order_by_id",
                description="Query GraphQL service for a sales order by ID",
                inputSchema={
                    "type": "object",
                    "properties": {"id": {"type": "string"}},
                    "required": ["id"]
                }
            ),
            types.Tool(
                name="get_employee_by_id",
                description="Query GraphQL service for an employee by ID",
                inputSchema={
                    "type": "object",
                    "properties": {"id": {"type": "string"}},
                    "required": ["id"]
                }
            )
        ]
    
    @server.call_tool()
    async def handle_call_tool(name: str, arguments: dict) -> list[types.TextContent]:
        if name == "list_sales_order":
            result = list_sales_order(graphql_client)
        elif name == "list_employees":
            result = list_employees(graphql_client)
        elif name == "get_sales_order_by_id":
            result = get_sales_order_by_id(graphql_client, arguments.get("id"))
        elif name == "get_employee_by_id":
            result = get_employee_by_id(graphql_client, arguments.get("id"))
        else:
            result = {"error": f"Unknown tool: {name}"}
        
        return [types.TextContent(type="text", text=json.dumps(result))]
    
    # 5. Start MCP server
    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            InitializationOptions(
                server_name="mcp-graphql-server",
                server_version="0.1.0"
            )
        )
```

**Error handling:**

- Missing `GRAPHQL_ENDPOINT` → log error and exit with status 1
- Invalid endpoint URL (caught during `GraphQLClient.__init__`) → log error and exit with status 1
- Tool invocation errors are handled by individual tool handlers (return `{"error": "..."}` dict)
- Unknown tool names → return `{"error": "Unknown tool: <name>"}`

**Dependencies:**

- MCP SDK for Python (`mcp` package) for protocol implementation
- `GraphQLClient` from `mcp.graphql_client`
- All four tool handlers imported from `mcp.tools.*`
- Python standard library: `os`, `sys`, `logging`, `json`

---

### Algorithm Notes

#### Build Order

The story is implemented through six incremental steps:

1. **GraphQL Client** (`mcp/graphql_client.py` + `tests/test_graphql_client.py`)
   - Implement `NetworkError` exception class
   - Implement `GraphQLClient.__init__` with endpoint validation
   - Implement `GraphQLClient.execute_query` with HTTP POST, JSON parsing, error extraction
   - Write 4 unit tests (success, HTTP error, network error, malformed JSON)

2. **Tool: `list_sales_order`** (`mcp/tools/list_sales_order.py` + `tests/tools/test_list_sales_order.py`)
   - Implement handler function with GraphQL query construction
   - Implement error handling (NetworkError, ValueError, GraphQL errors, missing data)
   - Write 2 unit tests (success with data, GraphQL error response)

3. **Tool: `list_employees`** (`mcp/tools/list_employees.py` + `tests/tools/test_list_employees.py`)
   - Implement handler function with GraphQL query construction
   - Implement error handling (NetworkError, ValueError, GraphQL errors, missing data)
   - Write 2 unit tests (success with data, GraphQL error response)

4. **Tool: `get_sales_order_by_id`** (`mcp/tools/get_sales_order_by_id.py` + `tests/tools/test_get_sales_order_by_id.py`)
   - Implement handler function with input validation and GraphQL query construction
   - Implement error handling (missing ID, NetworkError, ValueError, GraphQL errors, missing data)
   - Write 4 unit tests (success found, missing ID, not found, GraphQL error response)

5. **Tool: `get_employee_by_id`** (`mcp/tools/get_employee_by_id.py` + `tests/tools/test_get_employee_by_id.py`)
   - Implement handler function with input validation and GraphQL query construction
   - Implement error handling (missing ID, NetworkError, ValueError, GraphQL errors, missing data)
   - Write 4 unit tests (success found, missing ID, not found, GraphQL error response)

6. **MCP Server** (`mcp/server.py` + `tests/test_mcp_server.py`)
   - Implement `main()` function with environment variable reading, client initialization, tool registration
   - Implement MCP server startup and request handling
   - Write 2 integration tests (`list_sales_order` and `list_employees` against local GraphQL service)
   - Update `README.md` with setup instructions, usage examples, environment variable documentation

Each step is independently testable. Unit tests (steps 1-5) run with mocked dependencies. Integration tests (step 6) require the local GraphQL service from PER-5.

#### Error Message Format

All error messages follow consistent patterns:

- Input validation: `"Missing required parameter: <param>"`
- Network errors: `"GraphQL service unavailable: <message>"`
- HTTP errors: `"Failed to fetch <resource>: HTTP error <status>"`
- GraphQL errors: `"Failed to fetch <resource>: <first_error_message>"`
- Malformed responses: `"Invalid response from GraphQL service: <message>"`
- Missing data: `"Failed to fetch <resource>: no data returned"`

Where `<resource>` is one of: `sales orders`, `employees`, `sales order <id>`, `employee <id>`.

#### Test Data

**Unit tests (mocked GraphQL client):**

- Sales orders fixture: `[{"id": "SO-001", "customerId": "C-123", "total": 1500.0, "status": "shipped"}]`
- Employees fixture: `[{"id": "E-001", "name": "Alice Smith", "department": "Engineering", "email": "alice@example.com"}]`

**Integration tests (local GraphQL service):**

- Requires GraphQL service from PER-5 running at `GRAPHQL_ENDPOINT` URL
- Service must be seeded with at least one sales order and one employee
- Tests verify JSON shape and presence of data, not specific values

---

## Tests

This story includes 18 automated tests total that map 1:1 to the 14 acceptance criteria and verification approach in breakdown.md:

### Unit Tests (16 tests)

#### GraphQL Client Tests (`tests/test_graphql_client.py`) — 4 tests

1. **test_execute_query_success** — Valid GraphQL response with data and no errors returns `(data_dict, [])`
   - **Verifies:** GraphQL client successfully parses responses
   
2. **test_execute_query_http_error** — HTTP 500 status code returns `(None, [{"message": "HTTP error: 500"}])`
   - **Verifies:** AC2 (AC11) - error handling for unreachable service
   
3. **test_execute_query_network_error** — Connection refused raises `NetworkError` with message
   - **Verifies:** AC2 (AC11) - error handling for network failures
   
4. **test_execute_query_malformed_json** — Invalid JSON response raises `ValueError` with message
   - **Verifies:** AC3 (AC13) - error handling for malformed responses

#### Tool Handler Tests (12 tests with mocked GraphQL client)

##### `list_sales_order` (`tests/tools/test_list_sales_order.py`) — 2 tests

5. **test_list_sales_order_success** — Mocked client returns sales order data → tool returns `{"salesOrders": [...]}`
   - **Verifies:** AC2 (AC3) - correct JSON shape for list_sales_order success
   - **Verifies:** AC3 (AC12) - tool constructs correct GraphQL query
   
6. **test_list_sales_order_error** — Mocked client returns GraphQL error → tool returns `{"error": "Failed to fetch sales orders: <message>"}`
   - **Verifies:** AC2 (AC11) - consistent error shape
   - **Verifies:** AC3 (AC13) - error handling for GraphQL errors

##### `list_employees` (`tests/tools/test_list_employees.py`) — 2 tests

7. **test_list_employees_success** — Mocked client returns employee data → tool returns `{"employees": [...]}`
   - **Verifies:** AC2 (AC4) - correct JSON shape for list_employees success
   - **Verifies:** AC3 (AC12) - tool constructs correct GraphQL query
   
8. **test_list_employees_error** — Mocked client returns GraphQL error → tool returns `{"error": "Failed to fetch employees: <message>"}`
   - **Verifies:** AC2 (AC11) - consistent error shape
   - **Verifies:** AC3 (AC13) - error handling for GraphQL errors

##### `get_sales_order_by_id` (`tests/tools/test_get_sales_order_by_id.py`) — 4 tests

9. **test_get_sales_order_by_id_success** — Mocked client returns sales order → tool returns `{"salesOrder": {...}}`
   - **Verifies:** AC2 (AC5) - correct JSON shape when sales order found
   - **Verifies:** AC3 (AC12) - tool constructs correct GraphQL query with variables
   
10. **test_get_sales_order_by_id_missing_id** — No ID parameter → tool returns `{"error": "Missing required parameter: id"}`
    - **Verifies:** AC2 (AC6) - input validation for missing ID parameter
    
11. **test_get_sales_order_by_id_not_found** — Mocked client returns `({"salesOrder": None}, [])` → tool returns `{"salesOrder": null}`
    - **Verifies:** AC2 (AC7) - null result when sales order not found
    
12. **test_get_sales_order_by_id_error** — Mocked client returns GraphQL error → tool returns `{"error": "Failed to fetch sales order <id>: <message>"}`
    - **Verifies:** AC2 (AC11) - consistent error shape
    - **Verifies:** AC3 (AC13) - error handling for GraphQL errors

##### `get_employee_by_id` (`tests/tools/test_get_employee_by_id.py`) — 4 tests

13. **test_get_employee_by_id_success** — Mocked client returns employee → tool returns `{"employee": {...}}`
    - **Verifies:** AC2 (AC8) - correct JSON shape when employee found
    - **Verifies:** AC3 (AC12) - tool constructs correct GraphQL query with variables
    
14. **test_get_employee_by_id_missing_id** — No ID parameter → tool returns `{"error": "Missing required parameter: id"}`
    - **Verifies:** AC2 (AC9) - input validation for missing ID parameter
    
15. **test_get_employee_by_id_not_found** — Mocked client returns `({"employee": None}, [])` → tool returns `{"employee": null}`
    - **Verifies:** AC2 (AC10) - null result when employee not found
    
16. **test_get_employee_by_id_error** — Mocked client returns GraphQL error → tool returns `{"error": "Failed to fetch employee <id>: <message>"}`
    - **Verifies:** AC2 (AC11) - consistent error shape
    - **Verifies:** AC3 (AC13) - error handling for GraphQL errors

### Integration Tests (2 tests)

#### MCP Server Tests (`tests/test_mcp_server.py`) — 2 tests

17. **test_list_sales_order_integration** — Invoke `list_sales_order` tool against local GraphQL service → verify JSON shape `{"salesOrders": [...]}`
    - **Verifies:** AC1 (AC1, AC2) - MCP server exposes and invokes list_sales_order
    - **Verifies:** AC3 (AC14) - end-to-end query to live GraphQL service
    
18. **test_list_employees_integration** — Invoke `list_employees` tool against local GraphQL service → verify JSON shape `{"employees": [...]}`
    - **Verifies:** AC1 (AC1, AC2) - MCP server exposes and invokes list_employees
    - **Verifies:** AC3 (AC14) - end-to-end query to live GraphQL service

### Test Execution

**Unit tests only (no external dependencies):**

```bash
pytest tests/ -k "not integration"
```

Expected output: 16 tests passed

**Integration tests (requires local GraphQL service):**

```bash
export GRAPHQL_ENDPOINT=http://localhost:4000/graphql
pytest tests/test_mcp_server.py
```

Expected output: 2 tests passed

**All tests:**

```bash
export GRAPHQL_ENDPOINT=http://localhost:4000/graphql
pytest tests/
```

Expected output: 18 tests passed

---

## Dependencies

**External:**

- Python `requests` library for HTTP communication
- MCP SDK for Python (`mcp` package) for protocol implementation
- `pytest` for test execution
- `unittest.mock` (standard library) for mocking in unit tests
- GraphQL service from PER-5 (running at `GRAPHQL_ENDPOINT`) for integration tests

**Internal:**

- All tool handlers depend on `GraphQLClient`
- MCP server depends on all tool handlers and `Graph
