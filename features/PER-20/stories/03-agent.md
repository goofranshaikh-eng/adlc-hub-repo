# LLD: Story 03-agent — Implement FastMCP HTTP client and integrate tool calls

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 03-agent  
**After:** Story 02-agent (LLM client and orchestrator must exist)

---

## Acceptance Criteria

### AC-2: Tool calls over FastMCP HTTP

**Given** `MCP_URL` points at the FastAPI and FastMCP server  
**When** the LLM requests employee or sales-order data  
**Then** the agent calls that tool over FastMCP HTTP (not stdio) and returns the synthesized answer  
**And** the tools remain `list_sales_order`, `list_employees`, `get_sales_order_by_id`, and `get_employee_by_id`  
**And** ID arguments are normalized the way they are today (supported format determined by investigating original stdio MCP agent behavior; documented in `mcp_client.py` implementation)

### AC-3: MCP error handling

**Given** a tool returns `{"error": "..."}` or the MCP server is unreachable  
**When** the question needed that tool  
**Then** the agent returns a clear error and stays up  
**And** it does not start a stdio MCP process

### AC-4: Health check (MCP reachable when up)

**Given** the agent process is started  
**When** an operator checks health on the FastAPI app  
**Then** they can see the process is up and whether the MCP HTTP endpoint is reachable, without calling a tool

---

## File Plan

### New Files

- `agent/mcp_client.py` — FastMCP HTTP client; calls four MCP tools over SSE at `MCP_URL`
- `tests/test_tool_call.py` — AC-2: tool call over FastMCP SSE with successful result (covers all four tools)
- `tests/test_id_normalization.py` — AC-2: ID normalization (tests all historically supported formats determined by investigation)
- `tests/test_mcp_unreachable.py` — AC-3: MCP server unreachable
- `tests/test_tool_error.py` — AC-3: tool returns `{"error": "..."}`
- `tests/test_health_mcp_up.py` — AC-4: health endpoint when MCP server up

### Modified Files

- `agent/orchestrator.py` — Replace `mcp_stub.call_tool` with `mcp_client.call_tool`; add direct error return on tool failure (AC-3 fix)
- `agent/health.py` — Update `_probe_mcp` to use GET with `Accept: text/event-stream` header
- `requirements.txt` — Ensure `httpx` and `mcp` library are listed

### Removed Files

- `agent/mcp_stub.py` — Replaced by `mcp_client.py`
- Any Flask app entry point (e.g. `agent/flask_app.py`) — remove if present
- Any stdio MCP subprocess logic (e.g. `agent/stdio_mcp.py`) — remove if present

### No changes outside `agent` repo

---

## Interfaces

### FastMCP HTTP Client Interface (`agent/mcp_client.py`)

**Function:** `async def call_tool(tool_name: str, tool_args: dict) -> dict`

**Input:**
- `tool_name` (string): One of `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`
- `tool_args` (dict): Tool arguments (e.g. `{"id": "emp-001"}`)

**Output:**
```json
{
  "result": <tool result data>
}
```
or
```json
{
  "error": "string"
}
```

**Behavior:**
- **ID normalization:** Investigate original stdio MCP agent code (if discoverable in repo history, docs, or related repos) to determine what ID formats were historically supported. Implement matching normalization logic to preserve existing behavior. If investigation yields no evidence of prior normalization, document the assumption and implement minimal normalization (uppercase prefix for hyphenated IDs like `emp-001` → `EMP-001`). Document the actual supported formats in `_normalize_id()` docstring based on investigation findings.
- Call MCP tool over FastMCP SSE at `MCP_URL` using `mcp` library's `sse_client`
- On success, return tool result wrapped in `{"result": ...}`
- On tool error (MCP returns error), return `{"error": "..."}`
- On MCP server unreachable (connection error, timeout), return `{"error": "MCP server unreachable"}`
- Do NOT fall back to stdio subprocess (AC-3)

**Environment:**
- `MCP_URL` (required): FastMCP SSE URL (e.g. `http://127.0.0.1:8000/sse`)

**Transport:**
- FastMCP SSE protocol using `mcp` library's `sse_client` and `ClientSession`
- Timeout: 10 seconds (fail fast if MCP server slow or down)

---

### Updated Orchestrator Interface (`agent/orchestrator.py`)

**Changes:**
- Replace `from agent.mcp_stub import call_tool` with `from agent.mcp_client import call_tool`
- **AC-3 fix:** Check `if "error" in tool_result: return {"error": tool_result["error"]}` to return tool errors directly without LLM synthesis

**Behavior:**
- AC-1: Direct answer → return `{"answer": "..."}`
- AC-2: Tool success → LLM synthesizes result → return `{"answer": "..."}`
- **AC-3: Tool error → return `{"error": "..."}` directly (no synthesis)**
- AC-3: LLM unavailable → return `{"error": "LLM unavailable"}`

---

### Updated Health Endpoint Interface (`agent/health.py`)

**Changes:**
- Update `_probe_mcp` to use GET with `Accept: text/event-stream` header instead of HEAD

**Behavior:**
- `GET /health` → `{"status": "up", "mcp_reachable": true}` when MCP server up
- `mcp_reachable: false` when `MCP_URL` unset or MCP server unreachable

---

## Algorithm Notes

### ID Normalization Investigation (Pre-Implementation)

**Objective:** Determine what ID formats the original stdio MCP agent supported to preserve existing behavior (AC-2 exception criterion: "ID arguments are normalized the way they are today").

**Investigation steps:**

1. **Search repo history:** Check git log for prior `agent/` files mentioning "normalize", "ID", or "stdio_mcp". Look for deleted files in git history.
2. **Check related repos:** If `adlc-mcp-server` or `adlc-graphql` repos contain agent client code or documentation, examine those.
3. **Review requirements/tickets:** Search epic/story docs for prior ID format examples (e.g., do they show `emp001` without hyphen, or only `emp-001`?).
4. **Examine existing tests:** If any tests reference ID formats, note the patterns.
5. **Document findings:** In `mcp_client.py` docstring, list discovered supported formats with examples. If no evidence found, document assumption and implement minimal normalization.

**Fallback if investigation yields nothing:**
- **Assumption:** Only hyphenated IDs were normalized (`emp-001` → `EMP-001`). Non-hyphenated IDs (`emp001`) were not supported or were passed through as-is.
- **Document this assumption** in `_normalize_id()` docstring.
- **Recommendation:** If production logs or user reports later show non-hyphenated IDs were used, file a follow-up ticket to restore that support.

---

### `agent/mcp_client.py` (New)

**Responsibilities:**
- Call four MCP tools over FastMCP SSE at `MCP_URL` using `mcp` library
- Normalize `id` arguments before call (matching original stdio MCP agent behavior; documented based on investigation)
- Handle connection errors (MCP unreachable) and tool errors
- No stdio fallback

**Pseudocode:**
```python
import os
import re
from mcp import ClientSession
from mcp.client.sse import sse_client

_MCP_URL = os.getenv("MCP_URL", "")

async def call_tool(tool_name: str, tool_args: dict) -> dict:
    """
    Call MCP tool over FastMCP SSE.
    
    Args:
        tool_name: One of list_sales_order, list_employees, 
                   get_sales_order_by_id, get_employee_by_id
        tool_args: Tool arguments (e.g. {"id": "emp-001"})
    
    Returns:
        {"result": <data>} on success
        {"error": "..."} on failure
    
    Notes:
        ID normalization: See _normalize_id() for supported formats.
        Implementation preserves original stdio MCP agent behavior 
        (documented based on investigation; see function docstring).
    """
    if not _MCP_URL:
        return {"error": "MCP_URL not configured"}
    
    # Normalize ID argument if present (matches original stdio behavior)
    normalized_args = _normalize_id(tool_args)
    
    try:
        # Use FastMCP SSE client from mcp library
        async with sse_client(_MCP_URL) as (read, write):
            async with ClientSession(read, write) as session:
                # Initialize session
                await session.initialize()
                
                # Call tool
                result = await session.call_tool(tool_name, normalized_args)
                
                # Check for error in result
                if hasattr(result, 'isError') and result.isError:
                    error_content = getattr(result, 'content', [])
                    error_msg = error_content[0].text if error_content else 'Tool error'
                    return {"error": error_msg}
                
                # Extract result content
                if hasattr(result, 'content') and result.content:
                    return {"result": result.content[0].text}
                
                return {"result": result}
    
    except ConnectionError:
        return {"error": "MCP server unreachable"}
    except TimeoutError:
        return {"error": "MCP server timeout"}
    except Exception as e:
        return {"error": f"MCP client error: {str(e)}"}

def _normalize_id(args: dict) -> dict:
    """
    Normalize 'id' argument to match original stdio MCP agent behavior.
    
    **Investigation findings:** [TO BE FILLED DURING IMPLEMENTATION]
    - If original agent code was found: Document discovered patterns here.
    - If no evidence found: Document assumption below.
    
    **Assumed supported format (if no evidence found):**
    Single alphabetic prefix followed by hyphen and digits (case-insensitive).
    Examples: emp-001 → EMP-001, so-123 → SO-123.
    
    **NOT normalized (passed through as-is):**
    - IDs without hyphen (e.g., emp001) — no evidence of support
    - Multi-part prefixes (e.g., foo-bar-123) — no evidence of support
    
    **Regex pattern (assumed):** ^([a-z]+)-(\d+)$
    - Matches: emp-001, EMP-001, so-123, SO-123
    - Does NOT match: emp001 (no hyphen), foo-bar-123 (multi-part)
    
    If production evidence shows non-hyphenated IDs were used historically,
    file a follow-up ticket to restore that support.
    
    Args:
        args: Tool arguments dict
    
    Returns:
        Dict with normalized id if present and format matches, else unchanged
    """
    if "id" not in args:
        return args
    
    id_value = args["id"]
    
    # Pattern: <single-alphabetic-prefix>-<digits>
    # (Adjust regex based on investigation findings; this is the fallback)
    match = re.match(r'^([a-z]+)-(\d+)$', id_value, re.IGNORECASE)
    if match:
        prefix, digits = match.groups()
        normalized_id = f"{prefix.upper()}-{digits}"
        return {**args, "id": normalized_id}
    
    # No match: return as-is (unsupported format or no normalization needed)
    return args
```

**Implementation notes:**

- **Before coding `_normalize_id()`:** Follow investigation steps above. Update docstring with findings.
- **If evidence found of non-hyphenated ID support:** Adjust regex to `^([a-z]+)-?(\d+)$` (hyphen optional) and update docstring examples.
- **If evidence found of other patterns (e.g., multi-part prefixes):** Adjust regex and document.
- **If no evidence found:** Keep regex as-is (`^([a-z]+)-(\d+)$`), document assumption in docstring, and note in PR that follow-up investigation is recommended if production logs show non-hyphenated IDs.

---

### `agent/orchestrator.py` (Modified)

**Changes:**
- Replace import: `from agent.mcp_stub import call_tool` → `from agent.mcp_client import call_tool`
- **AC-3 fix:** Add `if "error" in tool_result: return {"error": tool_result["error"]}` after tool call to return errors directly without LLM synthesis

**Updated Pseudocode:**
```python
from agent.llm_client import ask_llm
from agent.mcp_client import call_tool  # CHANGED: was mcp_stub

async def process_question(question: str) -> dict:
    try:
        # Step 1: Ask LLM (no tool result yet)
        llm_response = await ask_llm(question)
        
        if llm_response["type"] == "direct_answer":
            # AC-1: Direct answer, no MCP call
            return {"answer": llm_response["answer"]}
        
        elif llm_response["type"] == "tool_request":
            # AC-2: Route to FastMCP SSE client
            tool_name = llm_response["tool_name"]
            tool_args = llm_response["tool_args"]
            
            tool_result = await call_tool(tool_name, tool_args)
            
            # AC-3 FIX: If tool returned error, return it directly (no synthesis)
            if "error" in tool_result:
                return {"error": tool_result["error"]}
            
            # Tool returned success data: synthesize into answer
            synthesized = await ask_llm(question, tool_result=tool_result)
            return {"answer": synthesized["answer"]}
    
    except Exception as e:
        # AC-3: LLM unavailable or unexpected error
        return {"error": "LLM unavailable"}
```

---

### `agent/health.py` (Modified)

**Changes:**
- Replace HEAD request with GET using `Accept: text/event-stream` header

**Updated Pseudocode:**
```python
import os
import httpx
from datetime import datetime, timedelta

_probe_cache = {"reachable": False, "timestamp": None}
_CACHE_TTL_SECONDS = 10

def check_health() -> dict:
    mcp_url = os.getenv("MCP_URL", "")
    
    if not mcp_url:
        return {"status": "up", "mcp_reachable": False}
    
    # Check cache
    now = datetime.now()
    if _probe_cache["timestamp"] and (now - _probe_cache["timestamp"]) < timedelta(seconds=_CACHE_TTL_SECONDS):
        return {"status": "up", "mcp_reachable": _probe_cache["reachable"]}
    
    # Probe MCP endpoint
    reachable = _probe_mcp(mcp_url)
    _probe_cache["reachable"] = reachable
    _probe_cache["timestamp"] = now
    
    return {"status": "up", "mcp_reachable": reachable}

def _probe_mcp(url: str) -> bool:
    """
    Probe FastMCP SSE endpoint with GET request and Accept: text/event-stream header.
    
    FastMCP SSE endpoints do not respond to HEAD requests; use GET instead.
    """
    try:
        with httpx.Client(timeout=2.0) as client:
            response = client.get(
                url,
                headers={"Accept": "text/event-stream"}
            )
            # Accept any non-5xx response (2xx, 3xx, 4xx indicate server is up)
            return response.status_code < 500
    except Exception:
        return False
```

---

## Tests

### `tests/test_tool_call.py`

**Coverage:** AC-2 (tool call over FastMCP SSE with successful result — all four tools)

**Test 1: List sales orders over FastMCP SSE**

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock, MagicMock

def test_tool_call_list_sales_order(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    # Mock FastMCP SSE client and session
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='[{"id": "SO-001", "amount": 100}]')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        # LLM requests tool, then synthesizes result
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "list_sales_order", "tool_args": {}},
            {"type": "direct_answer", "answer": "Here are the sales orders: SO-001 ($100)"}
        ]
        
        response = client.post("/query", json={"question": "list sales orders"})
        
        assert response.status_code == 200
        assert "sales orders" in response.json()["answer"].lower()
        
        # Verify FastMCP SSE client used
        mock_session.call_tool.assert_called_once_with("list_sales_order", {})
```

**Test 2: List employees over FastMCP SSE**

**Pseudocode:**
```python
def test_tool_call_list_employees(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='[{"id": "EMP-001", "name": "Alice"}]')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "list_employees", "tool_args": {}},
            {"type": "direct_answer", "answer": "Here are the employees: Alice"}
        ]
        
        response = client.post("/query", json={"question": "list employees"})
        
        assert response.status_code == 200
        assert "employees" in response.json()["answer"].lower()
        
        mock_session.call_tool.assert_called_once_with("list_employees", {})
```

**Test 3: Get employee by ID over FastMCP SSE**

**Pseudocode:**
```python
def test_tool_call_get_employee_by_id(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='{"id": "EMP-001", "name": "Alice"}')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_employee_by_id", "tool_args": {"id": "EMP-001"}},
            {"type": "direct_answer", "answer": "Employee EMP-001 is Alice"}
        ]
        
        response = client.post("/query", json={"question": "get employee emp-001"})
        
        assert response.status_code == 200
        assert "Alice" in response.json()["answer"]
        
        mock_session.call_tool.assert_called_once_with("get_employee_by_id", {"id": "EMP-001"})
```

**Test 4: Get sales order by ID over FastMCP SSE**

**Pseudocode:**
```python
def test_tool_call_get_sales_order_by_id(monkeypatch):
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='{"id": "SO-123", "amount": 250}')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_sales_order_by_id", "tool_args": {"id": "SO-123"}},
            {"type": "direct_answer", "answer": "Sales order SO-123 is $250"}
        ]
        
        response = client.post("/query", json={"question": "get sales order so-123"})
        
        assert response.status_code == 200
        assert "250" in response.json()["answer"]
        
        mock_session.call_tool.assert_called_once_with("get_sales_order_by_id", {"id": "SO-123"})
```

---

### `tests/test_id_normalization.py`

**Coverage:** AC-2 (ID normalization — all historically supported formats determined by investigation)

**Test 1: Hyphenated employee ID normalization (baseline)**

**Pseudocode:**
```python
from fastapi.testclient import TestClient
from agent.main import app
from unittest.mock import patch, AsyncMock, MagicMock

def test_id_normalization_hyphenated_employee(monkeypatch):
    """
    Verify ID normalization for hyphenated format (baseline).
    
    Example: emp-001 → EMP-001
    """
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='{"id": "EMP-001", "name": "Alice"}')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        # LLM requests tool with lowercase hyphenated id
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_employee_by_id", "tool_args": {"id": "emp-001"}},
            {"type": "direct_answer", "answer": "Employee EMP-001 is Alice"}
        ]
        
        response = client.post("/query", json={"question": "get employee emp-001"})
        
        assert response.status_code == 200
        assert "Alice" in response.json()["answer"]
        
        # Verify ID normalized to EMP-001 before FastMCP SSE call
        call_args = mock_session.call_tool.call_args
        assert call_args[0][0] == "get_employee_by_id"
        assert call_args[0][1]["id"] == "EMP-001"
```

**Test 2: Hyphenated sales order ID normalization (baseline)**

**Pseudocode:**
```python
def test_id_normalization_hyphenated_sales_order(monkeypatch):
    """
    Verify ID normalization for hyphenated format (baseline).
    
    Example: so-123 → SO-123
    """
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='{"id": "SO-123", "amount": 250}')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        # LLM requests tool with lowercase hyphenated id
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_sales_order_by_id", "tool_args": {"id": "so-123"}},
            {"type": "direct_answer", "answer": "Sales order SO-123 is $250"}
        ]
        
        response = client.post("/query", json={"question": "get sales order so-123"})
        
        assert response.status_code == 200
        assert "250" in response.json()["answer"]
        
        # Verify ID normalized to SO-123 before FastMCP SSE call
        call_args = mock_session.call_tool.call_args
        assert call_args[0][0] == "get_sales_order_by_id"
        assert call_args[0][1]["id"] == "SO-123"
```

**Test 3: Non-hyphenated ID format (conditional test)**

**Pseudocode:**
```python
def test_id_normalization_or_passthrough_non_hyphenated(monkeypatch):
    """
    Verify non-hyphenated ID format handling (conditional on investigation findings).
    
    If investigation found non-hyphenated IDs were normalized historically:
        Example: emp001 → EMP001 (or EMP-001 if hyphen was added)
    
    If no evidence found (fallback assumption):
        Example: emp001 → emp001 (passed through as-is, no normalization)
    
    This test MUST match the _normalize_id() implementation based on investigation.
    """
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    # Expected ID in result depends on investigation findings:
    # - If original normalized non-hyphenated: "EMP001" or "EMP-001"
    # - If no evidence (fallback): "emp001" (unchanged)
    mock_result.content = [MagicMock(text='{"id": "emp001", "name": "Bob"}')]  # Adjust based on findings
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        # LLM requests tool with non-hyphenated ID
        mock_llm.side_effect = [
            {"type": "tool_request", "tool_name": "get_employee_by_id", "tool_args": {"id": "emp001"}},
            {"type": "direct_answer", "answer": "Employee emp001 is Bob"}
        ]
        
        response = client.post("/query", json={"question": "get employee emp001"})
        
        assert response.status_code == 200
        assert "Bob" in response.json()["answer"]
        
        # Verify ID handling based on investigation findings:
        call_args = mock_session.call_tool.call_args
        assert call_args[0][0] == "get_employee_by_id"
        
        # ADJUST THIS ASSERTION BASED ON INVESTIGATION FINDINGS:
        # Option A (if original normalized non-hyphenated): assert call_args[0][1]["id"] == "EMP001"
        # Option B (if no evidence, fallback assumption): assert call_args[0][1]["id"] == "emp001"
        assert call_args[0][1]["id"] == "emp001"  # Fallback assumption; adjust after investigation
```

**Test 4: Multi-part prefix ID format (conditional test)**

**Pseudocode:**
```python
def test_id_multi_part_prefix_passthrough(monkeypatch):
    """
    Verify multi-part prefix ID format is passed through as-is (conditional on investigation).
    
    Example: foo-bar-123 → foo-bar-123 (no normalization unless investigation found otherwise)
    """
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    
    client = TestClient(app)
    
    mock_session = AsyncMock()
    mock_result = MagicMock()
    mock_result.isError = False
    mock_result.content = [MagicMock(text='{"id": "foo-bar-123", "name": "Charlie"}')]
    mock_session.call_tool.return_value = mock_result
    mock_session.initialize = AsyncMock()
    
    with patch("agent.llm_client.ask_llm", new_callable=AsyncMock) as mock_llm, \
         patch("agent.mcp_client.ClientSession", return_value=mock_session), \
         patch("agent.mcp_client.sse_client"):
        
        # LLM requests tool with multi-part prefix
