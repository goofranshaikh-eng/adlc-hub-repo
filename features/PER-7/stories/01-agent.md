# Story 01-agent: Low-Level Design

**Story ID:** 01-agent  
**Epic:** [agent] Agent: query MCP tools for sales orders and employees  
**Repository:** agent  
**Document Version:** 3.0  
**Last Updated:** 2025-01-20  
**Status:** Draft

---

## Overview

This story implements a complete agent that queries MCP tools for sales orders and employees. It includes the MCP client wrapper, agent core query processing, and verification, delivering all three acceptance criteria (AC1, AC2, AC3) in a single implementation slice.

---

## Acceptance Criteria

### AC1: Agent code exists
**Given** the agent implementation is complete  
**When** the agent codebase is reviewed  
**Then** agent code exists in the repository

### AC2: Agent can invoke all four MCP tools and return useful answers
**Given** the agent has access to the four MCP tools from PER-6  
**When** a user queries about sales orders or employees  
**Then** the agent invokes the appropriate MCP tool(s) and returns a useful answer

### AC3: Basic tests or documented manual verification script
**Given** the agent implementation is complete  
**When** verification is required  
**Then** either automated tests exist OR a documented manual verification script is provided

---

## File Plan

### New Files

| Path | Purpose | Key Exports |
|------|---------|-------------|
| `agent/src/mcp_client.py` | MCP tool invocation wrapper | `MCPClient` class with four tool methods |
| `agent/src/agent_core.py` | Agent orchestration and query processing | `AgentCore` class with `process_query()` method |
| `agent/tests/test_mcp_client.py` | Unit tests for MCP client | 4 test functions (one per tool) |
| `agent/tests/test_agent.py` | Integration tests for agent | 4 test functions (verification per AC3) |

### Modified Files

None (all new files).

---

## Interfaces

### MCP Client Class

**Location:** `agent/src/mcp_client.py`

**Class:** `MCPClient`

**Initialization:**
```python
__init__(server_url: str | None = None)
```
- Accepts optional `server_url` parameter (defaults to `MCP_SERVER_URL` environment variable)
- Validates endpoint is non-empty; raises `ValueError` if missing
- Stores endpoint for tool invocations

**Methods:**

#### `invoke_list_sales_order() → dict | None`
**Purpose:** Fetch all sales orders  
**Parameters:** None  
**Returns:** 
- Success: Dictionary with `data` key containing list of sales order records
- Error: `None`

#### `invoke_list_employees() → dict | None`
**Purpose:** Fetch all employees  
**Parameters:** None  
**Returns:**
- Success: Dictionary with `data` key containing list of employee records
- Error: `None`

#### `invoke_get_sales_order_by_id(order_id: str) → dict | None`
**Purpose:** Fetch single sales order by ID  
**Parameters:**  
- `order_id`: Sales order identifier (string)  
**Returns:**
- Success: Dictionary with `data` key containing single sales order record
- Error: `None` if not found or tool unavailable

#### `invoke_get_employee_by_id(employee_id: str) → dict | None`
**Purpose:** Fetch single employee by ID  
**Parameters:**  
- `employee_id`: Employee identifier (string)  
**Returns:**
- Success: Dictionary with `data` key containing single employee record
- Error: `None` if not found or tool unavailable

---

### Agent Core Class

**Location:** `agent/src/agent_core.py`

**Class:** `AgentCore`

**Initialization:**
```python
__init__(mcp_client: MCPClient)
```
- Accepts `MCPClient` instance for tool invocation
- No validation required (assumes valid client)

**Methods:**

#### `process_query(query: str) → str`
**Purpose:** Process natural language query and return formatted answer  
**Parameters:**  
- `query`: Natural language query string  
**Returns:** Human-readable answer string or error message

**Query Patterns Supported:**
- `"list sales orders"` → invokes `list_sales_order`
- `"list employees"` → invokes `list_employees`
- `"get sales order <id>"` → invokes `get_sales_order_by_id`
- `"get employee <id>"` → invokes `get_employee_by_id`

**Response Formats:**
- List queries: `"Found N items: <summary>"`
- Get-by-id queries: `"Item <name> (ID: <id>): [details]"`
- Errors: `"Error: <message>"`

---

## Algorithm Notes

### MCP Client Initialization
1. Accept optional `server_url` parameter
2. If `server_url` is `None`, read from `MCP_SERVER_URL` environment variable
3. Validate non-empty; raise `ValueError` with message "MCP server URL not configured" if missing
4. Store endpoint as instance attribute `self._server_url`

### Tool Invocation Pattern
1. Construct JSON-RPC request payload with tool name and parameters
2. Send HTTP POST to `self._server_url` with 10-second timeout
3. Parse response:
   - HTTP 200 + valid JSON with `result` key: return `result` value
   - HTTP error or `error` key in response: log warning, return `None`
4. Handle exceptions: network errors, JSON parse errors return `None`

### Query Processing
1. Parse query string using keyword matching:
   - Keywords `"list"` + `"sales"` → `list_sales_order`
   - Keywords `"list"` + `"employee"` → `list_employees`
   - Keywords `"get"` + `"sales"` + ID pattern → `get_sales_order_by_id`
   - Keywords `"get"` + `"employee"` + ID pattern → `get_employee_by_id`
2. Extract parameters (ID for get-by-id queries)
3. Invoke corresponding MCP client method
4. Format response:
   - List results: count + summary of IDs/names
   - Single entity: key details formatted as readable text
   - Tool returns `None`: return error message

---

## Data Structures

### MCP Request Payload (JSON-RPC)
```python
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": str,        # Tool name
    "arguments": dict   # Empty for list, {"id": str} for get-by-id
  },
  "id": int             # Unique request ID
}
```

### MCP Response (Success)
```python
{
  "jsonrpc": "2.0",
  "result": {
    "data": list | dict  # List of records or single record
  },
  "id": int
}
```

### MCP Response (Error)
```python
{
  "jsonrpc": "2.0",
  "error": {
    "code": int,
    "message": str
  },
  "id": int
}
```

---

## Implementation Details

### `agent/src/mcp_client.py`

**Imports:**
```python
import os
import logging
import requests
from typing import Optional

logger = logging.getLogger(__name__)
```

**Class Structure:**
```python
class MCPClient:
    def __init__(self, server_url: Optional[str] = None):
        if server_url is None:
            server_url = os.environ.get("MCP_SERVER_URL")
        
        if not server_url:
            raise ValueError("MCP server URL not configured")
        
        self._server_url = server_url
        self._request_id = 0
    
    def _invoke_tool(self, tool_name: str, params: dict) -> Optional[dict]:
        self._request_id += 1
        payload = {
            "jsonrpc": "2.0",
            "method": "tools/call",
            "params": {
                "name": tool_name,
                "arguments": params
            },
            "id": self._request_id
        }
        
        try:
            response = requests.post(
                self._server_url,
                json=payload,
                timeout=10
            )
            
            if response.status_code != 200:
                logger.warning(f"Tool {tool_name} returned HTTP {response.status_code}")
                return None
            
            data = response.json()
            
            if "error" in data:
                logger.warning(f"Tool {tool_name} returned error: {data['error']}")
                return None
            
            if "result" not in data:
                logger.warning(f"Tool {tool_name} returned malformed response")
                return None
            
            return data["result"]
        
        except requests.exceptions.RequestException as e:
            logger.warning(f"Network error calling tool {tool_name}: {e}")
            return None
        except (ValueError, KeyError) as e:
            logger.warning(f"Parse error for tool {tool_name}: {e}")
            return None
    
    def invoke_list_sales_order(self) -> Optional[dict]:
        return self._invoke_tool("list_sales_order", {})
    
    def invoke_list_employees(self) -> Optional[dict]:
        return self._invoke_tool("list_employees", {})
    
    def invoke_get_sales_order_by_id(self, order_id: str) -> Optional[dict]:
        return self._invoke_tool("get_sales_order_by_id", {"id": order_id})
    
    def invoke_get_employee_by_id(self, employee_id: str) -> Optional[dict]:
        return self._invoke_tool("get_employee_by_id", {"id": employee_id})
```

---

### `agent/src/agent_core.py`

**Imports:**
```python
import re
import logging
from typing import Optional
from agent.src.mcp_client import MCPClient

logger = logging.getLogger(__name__)
```

**Class Structure:**
```python
class AgentCore:
    def __init__(self, mcp_client: MCPClient):
        self._mcp_client = mcp_client
    
    def process_query(self, query: str) -> str:
        query_lower = query.lower().strip()
        
        # Parse intent and extract parameters
        if "list" in query_lower and "sales" in query_lower:
            return self._handle_list_sales_orders()
        
        elif "list" in query_lower and "employee" in query_lower:
            return self._handle_list_employees()
        
        elif "get" in query_lower and "sales" in query_lower:
            order_id = self._extract_id(query_lower)
            if not order_id:
                return "Error: No sales order ID provided. Use 'get sales order <id>'"
            return self._handle_get_sales_order(order_id)
        
        elif "get" in query_lower and "employee" in query_lower:
            employee_id = self._extract_id(query_lower)
            if not employee_id:
                return "Error: No employee ID provided. Use 'get employee <id>'"
            return self._handle_get_employee(employee_id)
        
        else:
            return "Error: Unable to understand query. Try 'list sales orders', 'list employees', 'get sales order <id>', or 'get employee <id>'"
    
    def _extract_id(self, query: str) -> Optional[str]:
        # Extract ID from query using regex (alphanumeric + hyphens)
        match = re.search(r'\b([a-z0-9\-]+)\s*$', query)
        return match.group(1) if match else None
    
    def _handle_list_sales_orders(self) -> str:
        result = self._mcp_client.invoke_list_sales_order()
        
        if result is None:
            return "Error: Unable to fetch sales orders"
        
        data = result.get("data", [])
        if not data:
            return "No sales orders found"
        
        count = len(data)
        ids = [item.get("id", "unknown") for item in data[:5]]
        summary = ", ".join(ids)
        if count > 5:
            summary += f", ... ({count - 5} more)"
        
        return f"Found {count} sales orders: {summary}"
    
    def _handle_list_employees(self) -> str:
        result = self._mcp_client.invoke_list_employees()
        
        if result is None:
            return "Error: Unable to fetch employees"
        
        data = result.get("data", [])
        if not data:
            return "No employees found"
        
        count = len(data)
        names = [item.get("name", "unknown") for item in data[:5]]
        summary = ", ".join(names)
        if count > 5:
            summary += f", ... ({count - 5} more)"
        
        return f"Found {count} employees: {summary}"
    
    def _handle_get_sales_order(self, order_id: str) -> str:
        result = self._mcp_client.invoke_get_sales_order_by_id(order_id)
        
        if result is None:
            return f"Error: Sales order {order_id} not found"
        
        data = result.get("data", {})
        order_id_val = data.get("id", "unknown")
        amount = data.get("amount", "unknown")
        
        return f"Sales Order {order_id_val}: Amount: {amount}"
    
    def _handle_get_employee(self, employee_id: str) -> str:
        result = self._mcp_client.invoke_get_employee_by_id(employee_id)
        
        if result is None:
            return f"Error: Employee {employee_id} not found"
        
        data = result.get("data", {})
        name = data.get("name", "unknown")
        emp_id = data.get("id", "unknown")
        department = data.get("department", "unknown")
        
        return f"Employee {name} (ID: {emp_id}): Department: {department}"
```

---

### `agent/tests/test_mcp_client.py`

**Imports:**
```python
import pytest
from unittest.mock import Mock, patch
from agent.src.mcp_client import MCPClient
```

**Test Cases:**

```python
@patch('agent.src.mcp_client.requests.post')
def test_invoke_list_sales_order_success(mock_post):
    """Test successful invocation of list_sales_order tool"""
    # Arrange
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "jsonrpc": "2.0",
        "result": {
            "data": [
                {"id": "SO-1", "amount": 1000},
                {"id": "SO-2", "amount": 2000}
            ]
        },
        "id": 1
    }
    mock_post.return_value = mock_response
    
    client = MCPClient(server_url="http://test-server")
    
    # Act
    result = client.invoke_list_sales_order()
    
    # Assert
    assert result is not None
    assert "data" in result
    assert len(result["data"]) == 2
    
    # Verify request payload
    call_args = mock_post.call_args
    payload = call_args[1]["json"]
    assert payload["params"]["name"] == "list_sales_order"
    assert payload["params"]["arguments"] == {}


@patch('agent.src.mcp_client.requests.post')
def test_invoke_list_employees_success(mock_post):
    """Test successful invocation of list_employees tool"""
    # Arrange
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "jsonrpc": "2.0",
        "result": {
            "data": [
                {"id": "E-1", "name": "Alice", "department": "Engineering"},
                {"id": "E-2", "name": "Bob", "department": "Sales"}
            ]
        },
        "id": 1
    }
    mock_post.return_value = mock_response
    
    client = MCPClient(server_url="http://test-server")
    
    # Act
    result = client.invoke_list_employees()
    
    # Assert
    assert result is not None
    assert "data" in result
    assert len(result["data"]) == 2
    
    # Verify request payload
    call_args = mock_post.call_args
    payload = call_args[1]["json"]
    assert payload["params"]["name"] == "list_employees"
    assert payload["params"]["arguments"] == {}


@patch('agent.src.mcp_client.requests.post')
def test_invoke_get_sales_order_by_id_success(mock_post):
    """Test successful invocation of get_sales_order_by_id tool"""
    # Arrange
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "jsonrpc": "2.0",
        "result": {
            "data": {"id": "SO-1", "amount": 1000, "status": "completed"}
        },
        "id": 1
    }
    mock_post.return_value = mock_response
    
    client = MCPClient(server_url="http://test-server")
    
    # Act
    result = client.invoke_get_sales_order_by_id("SO-1")
    
    # Assert
    assert result is not None
    assert "data" in result
    assert result["data"]["id"] == "SO-1"
    
    # Verify request payload
    call_args = mock_post.call_args
    payload = call_args[1]["json"]
    assert payload["params"]["name"] == "get_sales_order_by_id"
    assert payload["params"]["arguments"] == {"id": "SO-1"}


@patch('agent.src.mcp_client.requests.post')
def test_invoke_get_employee_by_id_success(mock_post):
    """Test successful invocation of get_employee_by_id tool"""
    # Arrange
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "jsonrpc": "2.0",
        "result": {
            "data": {"id": "E-1", "name": "Alice", "department": "Engineering"}
        },
        "id": 1
    }
    mock_post.return_value = mock_response
    
    client = MCPClient(server_url="http://test-server")
    
    # Act
    result = client.invoke_get_employee_by_id("E-1")
    
    # Assert
    assert result is not None
    assert "data" in result
    assert result["data"]["id"] == "E-1"
    
    # Verify request payload
    call_args = mock_post.call_args
    payload = call_args[1]["json"]
    assert payload["params"]["name"] == "get_employee_by_id"
    assert payload["params"]["arguments"] == {"id": "E-1"}
```

---

### `agent/tests/test_agent.py`

**Imports:**
```python
import pytest
from unittest.mock import Mock
from agent.src.agent_core import AgentCore
from agent.src.mcp_client import MCPClient
```

**Test Cases:**

```python
def test_verify_list_sales_order():
    """Test agent can invoke list_sales_order and return useful answer"""
    # Arrange
    mock_client = Mock(spec=MCPClient)
    mock_client.invoke_list_sales_order.return_value = {
        "data": [
            {"id": "SO-1", "amount": 1000},
            {"id": "SO-2", "amount": 2000},
            {"id": "SO-3", "amount": 3000}
        ]
    }
    
    agent = AgentCore(mcp_client=mock_client)
    
    # Act
    response = agent.process_query("list sales orders")
    
    # Assert
    assert "Found 3 sales orders" in response
    assert "SO-1" in response
    assert not response.startswith("Error")
    mock_client.invoke_list_sales_order.assert_called_once()


def test_verify_list_employees():
    """Test agent can invoke list_employees and return useful answer"""
    # Arrange
    mock_client = Mock(spec=MCPClient)
    mock_client.invoke_list_employees.return_value = {
        "data": [
            {"id": "E-1", "name": "Alice", "department": "Engineering"},
            {"id": "E-2", "name": "Bob", "department": "Sales"}
        ]
    }
    
    agent = AgentCore(mcp_client=mock_client)
    
    # Act
    response = agent.process_query("list employees")
    
    # Assert
    assert "Found 2 employees" in response
    assert "Alice" in response
    assert not response.startswith("Error")
    mock_client.invoke_list_employees.assert_called_once()


def test_verify_get_sales_order_by_id():
    """Test agent can invoke get_sales_order_by_id and return useful answer"""
    # Arrange
    mock_client = Mock(spec=MCPClient)
    mock_client.invoke_get_sales_order_by_id.return_value = {
        "data": {"id": "SO-1", "amount": 1000, "status": "completed"}
    }
    
    agent = AgentCore(mcp_client=mock_client)
    
    # Act
    response = agent.process_query("get sales order SO-1")
    
    # Assert
    assert "Sales Order SO-1" in response
    assert "Amount: 1000" in response
    assert not response.startswith("Error")
    mock_client.invoke_get_sales_order_by_id.assert_called_once_with("SO-1")


def test_verify_get_employee_by_id():
    """Test agent can invoke get_employee_by_id and return useful answer"""
    # Arrange
    mock_client = Mock(spec=MCPClient)
    mock_client.invoke_get_employee_by_id.return_value = {
        "data": {"id": "E-1", "name": "Alice", "department": "Engineering"}
    }
    
    agent = AgentCore(mcp_client=mock_client)
    
    # Act
    response = agent.process_query("get employee E-1")
    
    # Assert
    assert "Employee Alice" in response
    assert "ID: E-1" in response
    assert "Department: Engineering" in response
    assert not response.startswith("Error")
    mock_client.invoke_get_employee_by_id.assert_called_once_with("E-1")
```

---

## Tests

### Test Coverage

**Test Strategy:** Unit tests for MCP client verify tool invocation; integration tests for agent verify all four tools invocable and responses formatted correctly.

**Total Test Count:** 8 tests (4 unit tests for MCP client + 4 integration tests for agent)

### Unit Tests (MCP Client)

| Test | Tool Verified | Verification Approach | AC Mapped |
|------|---------------|----------------------|-----------|
| `test_invoke_list_sales_order_success` | `list_sales_order` | Mock HTTP 200 with list data; assert return is dictionary with `data` key; verify tool name in request | AC1 |
| `test_invoke_list_employees_success` | `list_employees` | Mock HTTP 200 with list data; assert return is dictionary with `data` key; verify tool name in request | AC1 |
| `test_invoke_get_sales_order_by_id_success` | `get_sales_order_by_id` | Mock HTTP 200 with single record; assert return is dictionary with `data` key; verify tool name and ID in request | AC1 |
| `test_invoke_get_employee_by_id_success` | `get_employee_by_id` | Mock HTTP 200 with single record; assert return is dictionary with `data` key; verify tool name and ID in request | AC1 |

### Integration Tests (Agent)

| Test | Tool Verified | Verification Approach | AC Mapped |
|------|---------------|----------------------|-----------|
| `test_verify_list_sales_order` | `list_sales_order` | Mock MCP client response with 3 sales orders; process query; assert response format "Found 3 sales orders: ..."; verify non-error | AC2, AC3 |
| `test_verify_list_employees` | `list_employees` | Mock MCP client response with 2 employees; process query; assert response format "Found 2 employees: ..."; verify non-error | AC2, AC3 |
| `test_verify_get_sales_order_by_id` | `get_sales_order_by_id` | Mock MCP client response with single sales order; process query; assert response format "Sales Order SO-1: Amount: ..."; verify non-error | AC2, AC3 |
| `test_verify_get_employee_by_id` | `get_employee_by_id` | Mock MCP client response with single employee; process query; assert response format "Employee <name> (ID: E-1): Department: ..."; verify non-error | AC2, AC3 |

### Test Mapping to Acceptance Criteria

| AC | Verified By |
|----|-------------|
| AC1: Agent code exists | File existence check + 4 unit tests confirm MCP client provides all four tool methods |
| AC2: Agent can invoke all four MCP tools and return useful answers | 4 integration tests confirm all four tools invocable and responses formatted correctly |
| AC3: Basic tests or documented manual verification script | 8 automated tests satisfy "basic tests" requirement |

---

## Dependencies

### External
- **PER-6 (MCP Server Tools):** Must be running and accessible for integration testing
- **PER-5 (GraphQL Schemas):** Indirect dependency (tools use schemas, agent does not)
- **`requests` library:** Python HTTP client
- **`pytest` library:** Test framework

### Internal
- None (single story delivers all components)

---

## Configuration

### Environment Variables
- `MCP_SERVER_URL`: MCP server endpoint
  - **Required:** Yes (initialization raises `ValueError` if missing and not provided as parameter)
  - **Example:** `http://localhost:8000`
  - **Default:** None (must be set explicitly)

---

## Error Cases

| Error Condition | Handling | Return Value |
|-----------------|----------|--------------|
| `MCP_SERVER_URL` not set and no `server_url` parameter | Raise `ValueError` in initialization | N/A (exception) |
| Network error (connection refused, timeout) | Catch exception, log warning | `None` (MCP client) / `"Error: Unable to fetch..."` (agent) |
| HTTP error (4xx, 5xx) | Log warning with status code | `None` (MCP client) / error message (agent) |
| Malformed response | Log warning | `None` (MCP client) / error message (agent) |
| Tool returns error (e.g., not found) | Log warning | `None` (MCP client) / `"Error: <entity> <id> not found"` (agent) |
| Unknown query intent | Return helpful error | `"Error: Unable to understand query..."` |
| Missing ID in get-by-id query | Return helpful error | `"Error: No <entity> ID provided..."` |
| Empty list response | Return no-results message | `"No <entities> found"` |

---

## Notes

- **Single story delivers all three ACs:** AC1 (agent code exists), AC2 (invokes all four tools and returns useful answers), AC3 (basic tests)
- **Authentication:** Out of scope (assumed handled by MCP tools)
- **No retries:** MCP client returns `None` on first failure; agent returns error message to user
- **Logging:** All errors logged at WARNING level with tool name and error details
- **JSON-RPC format:** Follows standard JSON-RPC 2.0 specification
- **Query parsing:** Simple keyword matching (not LLM); deterministic and sufficient for four supported patterns
- **Response formatting:** Plain text strings (not structured data); human-readable per AC2 requirement for "useful answers"
- **Test count alignment:** 
  - Ticket: "Basic tests or a documented manual verification script"
  - Requirements.md Coverage Mapping: 8 total tests (4 MCP client unit tests + 4 agent integration tests)
  - Breakdown.md Story 1 verification: 8 tests
  - Tests section above: 8 tests listed
  - **Alignment:** ✅ Test count matches ticket, requirements, and breakdown exactly
- **All file paths under `agent/` subtree** (matching repo-tree context); stack is Python (matching project-context)
- **No invented acceptance criteria:** AC1, AC2, AC3 directly from ticket

---

**End of LLD**
