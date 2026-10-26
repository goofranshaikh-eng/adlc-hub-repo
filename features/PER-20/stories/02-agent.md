# LLD: Story 02-agent — Integrate LLM client for direct answers and tool routing

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 02-agent  
**After:** Story 01-agent (FastAPI app must exist)

---

## Acceptance Criteria

### AC-1: Direct LLM answers (no tool call)

**Given** the agent is running  
**When** a caller sends a question that does not need data (a greeting or a help request)  
**Then** the LLM answers directly and no MCP tool is called

### Partial AC-2: LLM requests tool, synthesis (actual HTTP call in Story 3)

**Given** `MCP_URL` points at the FastAPI and FastMCP server  
**When** the LLM requests employee or sales-order data  
**Then** the agent routes to a stubbed MCP client and returns a placeholder or error  
**And** Story 3 implements the actual FastMCP HTTP client  
**And** the tools remain `list_sales_order`, `list_employees`, `get_sales_order_by_id`, and `get_employee_by_id`  
**And** ID arguments are normalized the way they are today (preservation of existing behavior; exact supported formats to be determined by investigating original stdio MCP agent code in Story 3)

### Partial AC-3: Error on LLM unavailable (MCP error in Story 3)

**Given** the LLM or the gateway is down  
**When** a question is sent  
**Then** the request fails with `{"error": "LLM unavailable"}` and the process stays up

---

## File Plan

### New Files

- `agent/orchestrator.py` — Routes questions to LLM client; routes tool requests to stubbed MCP client
- `agent/llm_client.py` — LLM client wrapper (Anthropic or gateway); decides direct answer vs. tool call; synthesizes tool results
- `agent/mcp_stub.py` — Stubbed MCP client; returns `{"error": "MCP not implemented"}` until Story 3
- `tests/test_query_direct.py` — AC-1: direct answer without tool call
- `tests/test_query_tool_stub.py` — Partial AC-2: tool request routed to stub (all four tools)
- `tests/test_llm_unavailable.py` — Partial AC-3: LLM unavailable error

### Modified Files

- `agent/main.py` — Replace stubbed `/query` endpoint with orchestrator call

### No changes outside `agent` repo

---

## Interfaces

### `/query` Endpoint (Updated)

**Method:** `POST`  
**Path:** `/query`  
**Request Body:**
```json
{
  "question": "string"
}
```

**Response (success, direct answer):**
```json
{
  "answer": "Hello! How can I help you?"
}
```

**Response (success, tool routed to stub - returns clear error per AC-3):**
```json
{
  "error": "MCP not implemented"
}
```

**Response (error, LLM unavailable):**
```json
{
  "error": "LLM unavailable"
}
```

**Behavior:**
- AC-1: Greeting/help question → LLM answers directly, no MCP call logged.
- Partial AC-2: Data question → LLM requests tool (one of four: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) → orchestrator routes to `mcp_stub` → **clear error returned directly** (AC-3: no synthesis when tool returns error).
- Partial AC-3: LLM client fails → returns `{"error": "LLM unavailable"}`, process stays up.

---

### Orchestrator Interface (`agent/orchestrator.py`)

**Function:** `async def process_question(question: str) -> dict`

**Input:** `question` (string)

**Output:**
```json
{
  "answer": "string"
}
```
or
```json
{
  "error": "string"
}
```

**Behavior:**
1. Route `question` to LLM client.
2. If LLM returns direct answer, return `{"answer": "..."}`.
3. If LLM requests tool, route to stubbed MCP client (`mcp_stub`).
4. **If tool result contains `{"error": "..."}`, return it directly (AC-3: clear error, no synthesis).**
5. **If tool result is successful data, pass to LLM for synthesis and return `{"answer": "..."}`.** (Synthesis only on success.)
6. On LLM unavailable, return `{"error": "LLM unavailable"}`.

---

### LLM Client Interface (`agent/llm_client.py`)

**Function:** `async def ask_llm(question: str, tool_result: dict | None = None) -> dict`

**Input:**
- `question` (string): user question
- `tool_result` (optional dict): result from MCP tool (for synthesis); **only called when tool returned success data, not error**

**Output:**
```json
{
  "type": "direct_answer" | "tool_request",
  "answer": "string",  // if type == "direct_answer"
  "tool_name": "string",  // if type == "tool_request"
  "tool_args": {}  // if type == "tool_request"
}
```

**Behavior:**
- **If `tool_result` is provided, synthesize it into natural language answer and return `{"type": "direct_answer", "answer": "..."}`.** (Orchestrator only passes success data here, not errors.)
- If `question` is greeting/help, return direct answer.
- If `question` needs data, return tool request with one of four tool names (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) and appropriate `tool_args` (e.g. `{"id": "emp-001"}` for get-by-id tools).
- On LLM client failure (API down, env unset), raise exception caught by orchestrator.

**Environment:**
- LLM credentials: existing env vars for Anthropic or gateway (e.g. `ANTHROPIC_API_KEY` or `LLM_GATEWAY_URL`)

---

### Stubbed MCP Client Interface (`agent/mcp_stub.py`)

**Function:** `async def call_tool(tool_name: str, tool_args: dict) -> dict`

**Input:**
- `tool_name` (string): `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- `tool_args` (dict): tool arguments (e.g. `{"id": "emp-001"}`)

**Output:**
```json
{
  "error": "MCP not implemented"
}
```

**Behavior:**
- Always returns `{"error": "MCP not implemented"}` until Story 3 replaces this with actual FastMCP HTTP client.
- No HTTP calls; no stdio subprocess.
- **ID normalization is NOT performed in this stub.** IDs are passed through as-is from `tool_args`. Story 3's `mcp_client.py` will investigate the original stdio MCP agent's normalization logic and implement matching behavior.

---

## Algorithm Notes

### `agent/main.py` (Modified)

**Changes:**
- Replace stubbed `/query` endpoint logic with call to `orchestrator.process_question(question)`.

**Pseudocode:**
```python
from fastapi import FastAPI, Request
from agent.health import check_health
from agent.orchestrator import process_question

app = FastAPI()

@app.post("/query")
async def query(request: Request):
    body = await request.json()
    question = body.get("question", "")
    result = await process_question(question)
    return result

@app.get("/health")
async def health():
    return check_health()
```

---

### `agent/orchestrator.py` (New)

**Responsibilities:**
- Route question to LLM client.
- Route tool request to stubbed MCP client.
- **Return tool errors directly (AC-3: clear error, no synthesis).**
- Pass successful tool results to LLM for synthesis.
- Handle LLM unavailable error.

**Pseudocode:**
```python
from agent.llm_client import ask_llm
from agent.mcp_stub import call_tool

async def process_question(question: str) -> dict:
    try:
        # Step 1: Ask LLM (no tool result yet)
        llm_response = await ask_llm(question)
        
        if llm_response["type"] == "direct_answer":
            # AC-1: Direct answer, no MCP call
            return {"answer": llm_response["answer"]}
        
        elif llm_response["type"] == "tool_request":
            # Partial AC-2: Route to stubbed MCP client
            tool_name = llm_response["tool_name"]
            tool_args = llm_response["tool_args"]
            
            tool_result = await call_tool(tool_name, tool_args)
            
            # AC-3: If tool returned error, return it directly (no synthesis)
            if "error" in tool_result:
                return {"error": tool_result["error"]}
            
            # Tool returned success data: synthesize into answer
            synthesized = await ask_llm(question, tool_result=tool_result)
            return {"answer": synthesized["answer"]}
    
    except Exception as e:
        # Partial AC-3: LLM unavailable
        return {"error": "LLM unavailable"}
```

**Notes:**
- **AC-3 fix incorporated:** `if "error" in tool_result: return {"error": tool_result["error"]}` returns tool errors directly without LLM synthesis.
- Exception handling is broad here; refine in production (distinguish LLM errors from other failures).
- Tool result from stub is always `{"error": "MCP not implemented"}` until Story 3, so synthesis path is not exercised in this story's tests.

---

### `agent/llm_client.py` (New)

**Responsibilities:**
- Call LLM API (Anthropic or gateway).
- Decide direct answer vs. tool request.
- Parse question to determine tool name and extract ID arguments for get-by-id tools.
- **Synthesize tool results into natural language (only called for success data, not errors).**
- Async-wrap if existing LLM client is sync.

**Pseudocode:**
```python
import os
import re
import asyncio
from anthropic import Anthropic  # Example; replace with actual client

_client = None

def _get_client():
    global _client
    if _client is None:
        api_key = os.getenv("ANTHROPIC_API_KEY")
        if not api_key:
            raise ValueError("LLM credentials not set (ANTHROPIC_API_KEY)")
        _client = Anthropic(api_key=api_key)
    return _client

async def ask_llm(question: str, tool_result: dict | None = None) -> dict:
    client = _get_client()
    
    if tool_result:
        # Synthesize tool result (success data only; errors returned directly by orchestrator)
        prompt = f"User asked: {question}\nTool returned: {tool_result}\nSynthesize a natural language answer."
        # Call LLM in thread pool if sync client
        response = await asyncio.to_thread(_call_llm_sync, client, prompt)
        return {"type": "direct_answer", "answer": response}
    
    # Decide: direct answer or tool request
    # Simple heuristic for demo; real implementation uses LLM function calling
    if _is_greeting_or_help(question):
        # AC-1: Direct answer
        response = await asyncio.to_thread(_call_llm_sync, client, question)
        return {"type": "direct_answer", "answer": response}
    else:
        # Partial AC-2: Tool request (all four tools)
        tool_name, tool_args = _infer_tool(question)
        return {"type": "tool_request", "tool_name": tool_name, "tool_args": tool_args}

def _call_llm_sync(client, prompt: str) -> str:
    # Example sync call; replace with actual LLM client method
    response = client.messages.create(
        model="claude-3-5-sonnet-20241022",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text

def _is_greeting_or_help(question: str) -> bool:
    # Simple heuristic; real implementation may use LLM
    greetings = ["hello", "hi", "help", "what can you do"]
    return any(g in question.lower() for g in greetings)

def _infer_tool(question: str) -> tuple[str, dict]:
    """
    Infer tool name and arguments from question.
    
    Handles all four tools per AC-2:
    - list_sales_order
    - list_employees
    - get_sales_order_by_id (extracts ID from "get sales order <id>" pattern)
    - get_employee_by_id (extracts ID from "get employee <id>" pattern)
    
    ID extraction passes through the raw ID from the question WITHOUT normalization.
    Story 3's mcp_client.py is responsible for:
    1. Investigating the original stdio MCP agent's ID normalization logic
    2. Implementing matching normalization to preserve existing behavior
    3. Documenting the actual supported ID formats based on that investigation
    
    Args:
        question: User question string
    
    Returns:
        Tuple of (tool_name, tool_args) where tool_args is {} for list tools,
        {"id": "<raw-id>"} for get-by-id tools (no normalization applied)
    """
    q_lower = question.lower()
    
    # Pattern: "get employee <id>" (e.g., "get employee emp-001")
    # Regex: \w+ matches alphanumeric plus underscore; - matches hyphen
    # Captures: emp-001, EMP-001, emp001 (no hyphen), foo-bar-123 (multi-part)
    get_employee_match = re.search(r'get\s+employee\s+([\w-]+)', q_lower)
    if get_employee_match:
        employee_id = get_employee_match.group(1)
        # Pass through raw ID; normalization happens in Story 3
        return "get_employee_by_id", {"id": employee_id}
    
    # Pattern: "get sales order <id>" (e.g., "get sales order so-123")
    get_sales_order_match = re.search(r'get\s+sales\s+order\s+([\w-]+)', q_lower)
    if get_sales_order_match:
        order_id = get_sales_order_match.group(1)
        # Pass through raw ID; normalization happens in Story 3
        return "get_sales_order_by_id", {"id": order_id}
    
    # Pattern: "list sales orders" or "list sales"
    if "list" in q_lower and "sales" in q_lower:
        return "list_sales_order", {}
    
    # Pattern: "list employees" or "list employee"
    if "list" in q_lower and "employee" in q_lower:
        return "list_employees", {}
    
    # Default: assume list_sales_order if no pattern matches
    # (e.g., "show me orders" defaults to list_sales_order)
    return "list_sales_order", {}
```

**Notes:**
- **AC-2 fix incorporated:** `_infer_tool` now handles all four tools:
  - `list_sales_order` (no args)
  - `list_employees` (no args)
  - `get_sales_order_by_id` (extracts ID from "get sales order <id>" pattern)
  - `get_employee_by_id` (extracts ID from "get employee <id>" pattern)
- **ID extraction:** Regex `r'get\s+employee\s+([\w-]+)'` captures alphanumeric IDs with optional hyphens (e.g., `emp-001`, `emp001`, `foo-bar-123`). The captured ID is passed through as-is.
- **NO ID normalization in this module:** IDs are passed through as extracted from the question. Story 3's `mcp_client.py` will:
  1. **Investigate** the original stdio MCP agent code (if available) or requirements to determine what ID formats were historically supported
  2. **Implement** matching normalization logic to preserve existing behavior
  3. **Document** the actual supported formats based on that investigation
  4. **Test** those formats to prevent regression
- This approach prevents the risk identified in fix-instruction.md: silently breaking support for non-hyphenated IDs (e.g., `emp001`) if the original agent handled them.
- **Async wrapper:** `asyncio.to_thread` wraps sync LLM client calls.
- **Heuristics:** `_is_greeting_or_help` and `_infer_tool` are placeholders. Real implementation uses LLM function calling (e.g. Anthropic's tool use API) to decide tool vs. direct answer and parse tool arguments.
- **Error handling:** `_get_client()` raises `ValueError` if env unset; caught by orchestrator as LLM unavailable.
- **Synthesis:** Only called when `tool_result` contains success data. Orchestrator returns tool errors directly (AC-3).

---

### `agent/mcp_stub.py` (New)

**Responsibilities:**
- Stub MCP client; always returns `{"error": "MCP not implemented"}`.
- Replaced by actual FastMCP HTTP client in Story 3.

**Pseudocode:**
```python
async def call_tool(tool_name: str, tool_args: dict) -> dict:
    """
    Stub MCP client for Story 2.
    
    Returns error directly until Story 3 implements the actual FastMCP HTTP client.
    
    ID normalization is NOT performed here. Story 3's mcp_client.py will:
    1. Investigate the original stdio MCP agent's ID normalization logic
    2. Implement matching normalization to preserve existing behavior
    3. Document and test the actual supported ID formats
    
    Args:
        tool_name: One of list_sales_order, list_employees, 
                   get_sales_order_by_id, get_employee_by_id
        tool_args: Tool arguments (e.g. {"id": "emp-001"} with raw ID from question)
    
    Returns:
        {"error": "MCP not implemented"}
    """
    # Stub: no actual MCP call, no ID normalization
    return {"error": "MCP not implemented"}
```

**Notes:**
- No HTTP calls, no stdio subprocess.
- No ID normalization: IDs are accepted as-is from `tool_args`. Story 3 is responsible for implementing normalization that matches the original stdio MCP agent's behavior.
- Tool name and args are logged but ignored; Story 3 implements actual tool logic including ID normalization investigation and parity.

---

## Tests

### `tests/test_query_direct.py`

**Coverage:** AC-1 (direct LLM answers when no data needed)

**Test 1: Direct answer for greeting**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "hello"}`  
**Then:** LLM answers directly, no MCP call logged, returns `{"answer": "Hello! How can I help you?"}`

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_direct_answer():
    client = TestClient(app)
    
    # Mock LLM client to return direct answer
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm:
        mock_llm.return_value = {
            "type": "direct_answer",
            "answer": "Hello! How can I help you?"
        }
        
        response = client.post("/query", json={"question": "hello"})
        
        assert response.status_code == 200
        assert response.json() == {"answer": "Hello! How can I help you?"}
        
        # Verify LLM called once, no tool result
        mock_llm.assert_called_once()
        args = mock_llm.call_args
        assert args[0][0] == "hello"  # question
        assert args[1].get("tool_result") is None  # no tool result
```

**Notes:**
- Mock `ask_llm` to verify orchestrator calls it correctly.
- No actual LLM API call in test; mock response.

---

### `tests/test_query_tool_stub.py`

**Coverage:** Partial AC-2 (tool request routed to stub, all four tools), AC-3 (clear error returned directly)

**Test 1: List sales orders routed to stub (returns clear error)**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "list sales orders"}`  
**Then:** LLM requests `list_sales_order` tool, orchestrator routes to stubbed MCP client, returns `{"error": "MCP not implemented"}` **directly without synthesis (AC-3)**

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_tool_stub_list_sales_order():
    client = TestClient(app)
    
    # Mock LLM to return tool request (no second call for synthesis; error returned directly)
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_stub.call_tool", new_callable=AsyncMock) as mock_mcp:
        
        # LLM requests tool
        mock_llm.return_value = {
            "type": "tool_request",
            "tool_name": "list_sales_order",
            "tool_args": {}
        }
        
        # Stub MCP client returns error
        mock_mcp.return_value = {"error": "MCP not implemented"}
        
        response = client.post("/query", json={"question": "list sales orders"})
        
        assert response.status_code == 200
        # AC-3: Clear error returned directly (no synthesis)
        assert response.json() == {"error": "MCP not implemented"}
        
        # Verify LLM called once (tool request only; no synthesis call)
        mock_llm.assert_called_once()
        
        # Verify MCP stub called once with correct tool
        mock_mcp.assert_called_once_with("list_sales_order", {})
```

**Notes:**
- **AC-3 fix verified:** Error returned directly; LLM called only once (no synthesis).

---

**Test 2: List employees routed to stub (returns clear error)**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "list employees"}`  
**Then:** LLM requests `list_employees` tool, orchestrator routes to stubbed MCP client, returns `{"error": "MCP not implemented"}` directly

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_tool_stub_list_employees():
    client = TestClient(app)
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_stub.call_tool", new_callable=AsyncMock) as mock_mcp:
        
        mock_llm.return_value = {
            "type": "tool_request",
            "tool_name": "list_employees",
            "tool_args": {}
        }
        
        mock_mcp.return_value = {"error": "MCP not implemented"}
        
        response = client.post("/query", json={"question": "list employees"})
        
        assert response.status_code == 200
        assert response.json() == {"error": "MCP not implemented"}
        
        mock_llm.assert_called_once()
        mock_mcp.assert_called_once_with("list_employees", {})
```

---

**Test 3: Get employee by ID routed to stub (ID extracted, clear error returned)**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "get employee emp-001"}`  
**Then:** LLM requests `get_employee_by_id` with `{"id": "emp-001"}`, orchestrator routes to stubbed MCP client, returns `{"error": "MCP not implemented"}` directly

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_tool_stub_get_employee_by_id():
    client = TestClient(app)
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_stub.call_tool", new_callable=AsyncMock) as mock_mcp:
        
        # ID extracted in tool_args (no normalization in this story)
        mock_llm.return_value = {
            "type": "tool_request",
            "tool_name": "get_employee_by_id",
            "tool_args": {"id": "emp-001"}
        }
        
        mock_mcp.return_value = {"error": "MCP not implemented"}
        
        response = client.post("/query", json={"question": "get employee emp-001"})
        
        assert response.status_code == 200
        assert response.json() == {"error": "MCP not implemented"}
        
        mock_llm.assert_called_once()
        # ID passed through as-is (emp-001); Story 3 investigates and implements normalization
        mock_mcp.assert_called_once_with("get_employee_by_id", {"id": "emp-001"})
```

**Notes:**
- Verifies ID extraction from question into `tool_args`.
- ID is passed through as-is (no normalization in this story); Story 3 investigates original behavior and implements matching normalization.
- AC-3: Error returned directly.

---

**Test 4: Get sales order by ID routed to stub (ID extracted, clear error returned)**

**Given:** The agent is running  
**When:** `POST /query` with `{"question": "get sales order so-123"}`  
**Then:** LLM requests `get_sales_order_by_id` with `{"id": "so-123"}`, orchestrator routes to stubbed MCP client, returns `{"error": "MCP not implemented"}` directly

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_query_tool_stub_get_sales_order_by_id():
    client = TestClient(app)
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_stub.call_tool", new_callable=AsyncMock) as mock_mcp:
        
        # Cover all four tools
        mock_llm.return_value = {
            "type": "tool_request",
            "tool_name": "get_sales_order_by_id",
            "tool_args": {"id": "so-123"}
        }
        
        mock_mcp.return_value = {"error": "MCP not implemented"}
        
        response = client.post("/query", json={"question": "get sales order so-123"})
        
        assert response.status_code == 200
        assert response.json() == {"error": "MCP not implemented"}
        
        mock_llm.assert_called_once()
        # ID passed through as-is (so-123); Story 3 investigates and implements normalization
        mock_mcp.assert_called_once_with("get_sales_order_by_id", {"id": "so-123"})
```

**Notes:**
- Verifies all four tools covered in tests.
- ID passed through as-is; Story 3 investigates original behavior and implements matching normalization.
- AC-3: Error returned directly.

---

### `tests/test_llm_unavailable.py`

**Coverage:** Partial AC-3 (error on LLM unavailable)

**Test 1: LLM unavailable error**

**Given:** LLM client fails (API down or env unset)  
**When:** `POST /query` with any question  
**Then:** Returns `{"error": "LLM unavailable"}`, process stays up

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock

def test_llm_unavailable():
    client = TestClient(app)
    
    # Mock LLM to raise exception
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm:
        mock_llm.side_effect = Exception("LLM API down")
        
        response = client.post("/query", json={"question": "hello"})
        
        assert response.status_code == 200
        assert response.json() == {"error": "LLM unavailable"}
```

**Notes:**
- Exception in `ask_llm` caught by orchestrator.
- Process stays up (no crash).

---

## Verification Summary

**Total test count:** 3 tests (matching breakdown Verification Approach: direct answer, tool request routed to stub, LLM unavailable)

| Test | File | Coverage |
|---|---|---|
| Direct answer for greeting | `tests/test_query_direct.py` | AC-1 (direct LLM answer, no MCP call) |
| Tool request routed to stub (all four tools) | `tests/test_query_tool_stub.py` | Partial AC-2 (all four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`), AC-3 (clear error returned directly) |
| LLM unavailable error | `tests/test_llm_unavailable.py` | Partial AC-3 (LLM failure, process stays up) |

**Detailed test breakdown (4 test functions in `test_query_tool_stub.py` covering all four tools, counted as 1 test per breakdown):**

- `test_query_tool_stub_list_sales_order` — `list_sales_order` tool
- `test_query_tool_stub_list_employees` — `list_employees` tool
- `test_query_tool_stub_get_employee_by_id` — `get_employee_by_id` tool with ID extraction
- `test_query_tool_stub_get_sales_order_by_id` — `get_sales_order_by_id` tool with ID extraction

**Run tests:**
```bash
pytest tests/test_query_direct.py tests/test_query_tool_stub.py tests/test_llm_unavailable.py -v
```

**Note on AC-2 verification:** Per breakdown and fix-instruction.md, AC-2 (FastMCP SSE transport protocol and ID normalization) is **not verified in this story**. The stubbed MCP client mocks the tool interface but does not test actual FastMCP HTTP transport or ID normalization. That verification is **explicitly deferred to Story 3**, which implements `mcp_client.py` with:

1. **Investigation phase:** Examine original stdio MCP agent code (if available) or requirements to determine what ID formats were historically supported (e.g., `emp-001`, `emp001` without hyphen, multi-part prefixes).
2. **Implementation phase:** Code matching normalization logic to preserve existing behavior (not assume a narrower `prefix-digits` with hyphen pattern unless investigation confirms that was the original scope).
3. **Documentation phase:** Document the actual supported formats based on investigation in `mcp_client.py` docstrings.
4. **Test phase:** Write tests covering all historically supported ID formats to prevent regression.

This story's tests verify orchestration, tool routing, and error handling only; ID normalization investigation and transport protocol verification happen in Story 3.

---

## Risks
