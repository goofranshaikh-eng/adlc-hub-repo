# LLD: Story 04-agent — Update README to document FastAPI primary path and remove Flask/stdio

**Repo:** `agent` (`adlc-agents`, root `/home/goos/projects/adlc-agents`)  
**Epic:** Integrate Agent with MCP server  
**Story ID:** 04-agent  
**After:** Story 03-agent (FastAPI app, LLM client, and FastMCP HTTP client must be complete)

---

## Acceptance Criteria

### AC-5: Setup documentation

**Given** the `adlc-agents` README  
**When** a developer follows setup  
**Then** they run the agent with FastAPI, set `MCP_URL` to the FastMCP SSE URL, set the LLM env, and `POST` a question  
**And** HTTP MCP is the documented primary path  
**And** Flask and stdio are not documented  
**And** ID format is documented (supported format determined by Story 3's investigation of original stdio MCP agent behavior; documented here based on those findings)

---

## File Plan

### Modified Files

- `README.md` — Document FastAPI setup, `MCP_URL`, LLM credentials, run instructions, query example with ID normalization examples matching Story 3's findings; remove all Flask and stdio MCP references

### No New or Removed Files

This story only updates documentation.

---

## Interfaces

### README API (Documented Endpoints)

**Query Endpoint:** `POST /query`

**Request:**
```json
{
  "question": "string"
}
```

**Response (success):**
```json
{
  "answer": "string"
}
```

**Response (error):**
```json
{
  "error": "string"
}
```

**Health Endpoint:** `GET /health`

**Response:**
```json
{
  "status": "up",
  "mcp_reachable": true
}
```

---

## Algorithm Notes

### README Content Template

The README is a **complete replacement** of the existing file, preserving only License/Contributing/Changelog sections if they exist and do not reference Flask/stdio.

**ID Format Documentation:**

The "ID Format" subsection under Usage **must match Story 3's `_normalize_id()` implementation** based on its investigation of the original stdio MCP agent. The template below provides **placeholder text** that Story 4 Dev will replace with the actual supported formats after Story 3 completes:

```markdown
**ID Format:**

The agent normalizes employee and sales-order IDs before calling MCP tools. [**PLACEHOLDER: Replace with Story 3's documented supported formats from `mcp_client.py::_normalize_id()` docstring**]

**Example (baseline assumption, adjust per Story 3 findings):**
- Supported format: `prefix-digits` with hyphen (e.g., `emp-001`, `so-123`)
- Normalization: prefix uppercased (`emp-001` → `EMP-001`, `so-123` → `SO-123`)
- Other formats (e.g., `emp001` without hyphen, multi-part prefixes like `foo-bar-123`) are [**passed through as-is / also normalized — adjust per Story 3 findings**]

**If Story 3 found evidence of broader historical support (e.g., non-hyphenated IDs like `emp001` were normalized):**
- Document all discovered formats with examples
- Show normalization behavior for each (e.g., `emp001` → `EMP001` or `EMP-001` if hyphen added)

**If Story 3 found no evidence (fallback assumption):**
- Document only `prefix-digits` with hyphen
- Note that other formats are passed through unchanged
- Include recommendation to file follow-up ticket if production logs show non-hyphenated IDs were used
```

**Full README content (with placeholder for ID Format section):**

```markdown
# TriNet Agent

AI agent that answers questions about TriNet employee and sales-order data. The agent uses an LLM (Anthropic Claude or compatible gateway) to decide whether to answer directly or call MCP tools over HTTP, then synthesizes the final answer.

**Architecture:**
- **FastAPI** web server serving `/query` and `/health` endpoints
- **LLM client** for direct answers and synthesis
- **FastMCP HTTP client** to call MCP tools at a shared MCP server

---

## Prerequisites

- **Python 3.11+** (or version specified in `pyproject.toml` / `requirements.txt`)
- **MCP Server** running at a known URL (e.g. `http://127.0.0.1:8000/sse`)
  - See [PER-18](https://linear.app/personalgs/issue/PER-18/serve-mcp-graphql-tools-over-fastapi-fastmcp-http-replace-stdio) for MCP server setup
- **LLM credentials** (Anthropic API key or gateway URL)

---

## Installation

1. Clone the repository:
   ```bash
   git clone <repo-url>
   cd adlc-agents
   ```

2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

---

## Configuration

Set the following environment variables:

| Variable | Required | Description | Example |
|---|---|---|---|
| `MCP_URL` | Yes | FastMCP SSE endpoint URL | `http://127.0.0.1:8000/sse` |
| `ANTHROPIC_API_KEY` | Yes* | Anthropic API key for Claude LLM | `sk-ant-...` |
| `LLM_GATEWAY_URL` | Yes* | Alternative: LLM gateway URL if not using Anthropic directly | `http://llm-gateway:8080` |

_*Either `ANTHROPIC_API_KEY` or `LLM_GATEWAY_URL` is required, depending on your LLM setup._

**Example `.env` file:**
```bash
MCP_URL=http://127.0.0.1:8000/sse
ANTHROPIC_API_KEY=sk-ant-your-api-key-here
```

Load environment variables:
```bash
export $(cat .env | xargs)
```

---

## Running the Agent

Start the FastAPI server with `uvicorn`:

```bash
uvicorn agent.main:app --host 0.0.0.0 --port 8001
```

**Options:**
- `--reload` — auto-reload on code changes (development only)
- `--workers N` — run N worker processes (production)

The agent will be available at `http://localhost:8001`.

---

## Usage

### Query Endpoint

Send questions to the agent via `POST /query`:

```bash
curl -X POST http://localhost:8001/query \
  -H "Content-Type: application/json" \
  -d '{"question": "list sales orders"}'
```

**Response (success):**
```json
{
  "answer": "Here are the sales orders: SO-001 ($100), SO-002 ($250), ..."
}
```

**Response (error):**
```json
{
  "error": "MCP server unreachable"
}
```

**Example questions:**
- `"hello"` — Direct greeting (no MCP call)
- `"list sales orders"` — Calls `list_sales_order` tool
- `"list employees"` — Calls `list_employees` tool
- `"get employee emp-001"` — Calls `get_employee_by_id` tool (ID normalized per format below)
- `"get sales order so-123"` — Calls `get_sales_order_by_id` tool (ID normalized per format below)

**ID Format:**

The agent normalizes employee and sales-order IDs before calling MCP tools.

**[IMPLEMENTATION INSTRUCTION: Replace this entire section with content from Story 3's `agent/mcp_client.py::_normalize_id()` docstring after Story 3 completes. Copy the "Investigation findings" and format examples verbatim. Do NOT assume the baseline format below is correct; Story 3's investigation determines the actual supported formats.]**

**Baseline assumption (if Story 3 found no evidence of prior normalization):**
- Supported format: `prefix-digits` with hyphen (e.g., `emp-001`, `so-123`)
- The prefix is uppercased before the MCP call:
  - `emp-001` → `EMP-001`
  - `so-123` → `SO-123`
- Other ID formats (e.g., `emp001` without hyphen, multi-part prefixes like `foo-bar-123`) are passed through as-is (no normalization).

**If Story 3 found evidence of broader historical support:**
- [Document all discovered formats with examples from Story 3's investigation]
- [Show normalization behavior for each format]
- [Note any formats that were dropped or added in the migration]

**If you are implementing this story BEFORE Story 3 completes:**
- Leave the placeholder text above and file a blocker comment: "Blocked on Story 3 `_normalize_id()` investigation findings. Cannot document ID format until implementation determines actual supported formats."

### Health Check

Check agent and MCP server status:

```bash
curl http://localhost:8001/health
```

**Response:**
```json
{
  "status": "up",
  "mcp_reachable": true
}
```

- `status: "up"` — Agent process is running
- `mcp_reachable: true` — MCP server at `MCP_URL` is reachable
- `mcp_reachable: false` — MCP server is down or `MCP_URL` is not set

---

## Development

### Running Tests

```bash
pytest tests/ -v
```

**Test coverage:**
- Direct LLM answers (no MCP call)
- MCP tool calls over HTTP (all four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`)
- ID normalization (all historically supported formats determined by Story 3's investigation)
- Error handling (MCP unreachable, tool errors, LLM unavailable)
- Health endpoint

### Project Structure

```
adlc-agents/
├── agent/
│   ├── main.py            # FastAPI app entry point
│   ├── orchestrator.py    # Routes questions to LLM/MCP
│   ├── llm_client.py      # LLM client wrapper
│   ├── mcp_client.py      # FastMCP HTTP client
│   └── health.py          # Health check logic
├── tests/                 # Test suite
├── requirements.txt       # Python dependencies
└── README.md              # This file
```

---
```

**Removal criteria:**

Remove any line or section referencing:
- `flask`, `Flask`
- `stdio`, `subprocess`
- `local MCP`, `stdio_mcp`
- Flask-specific commands (`flask run`, `FLASK_APP`, etc.)
- stdio-specific env vars (`MCP_STDIO_PATH`, `MCP_LOCAL`, etc.)

**Preservation logic:**

If existing README contains License, Contributing, or Changelog sections that do NOT reference Flask/stdio, append them at the end of the new README.

---

## Tests

### `tests/test_readme_setup.py`

**Coverage:** AC-5 (README setup → successful query)

**Test 1: End-to-end README setup verification**

**Given:** A clean environment (no prior agent installation)  
**When:** A developer follows the README setup instructions step-by-step  
**Then:** They successfully run the agent and query it

**Pseudocode:**
```python
import subprocess
import time
import requests
import pytest

def test_readme_setup_end_to_end(tmp_path, monkeypatch):
    """
    Follow README setup from clean environment and verify successful query.
    
    This test runs in CI with a mock MCP server and mock LLM client.
    """
    # Setup: Start mock MCP server (or use existing test fixture)
    # (Deferred to CI infra; assumes mock MCP at http://127.0.0.1:8000/sse)
    
    # Set env vars per README
    monkeypatch.setenv("MCP_URL", "http://127.0.0.1:8000/sse")
    monkeypatch.setenv("ANTHROPIC_API_KEY", "sk-ant-test-key")
    
    # Start agent (assumes agent/ code is importable in CI)
    proc = subprocess.Popen(
        ["uvicorn", "agent.main:app", "--host", "0.0.0.0", "--port", "8001"],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    
    # Wait for startup
    time.sleep(2)
    
    try:
        # Query agent per README usage example
        response = requests.post(
            "http://localhost:8001/query",
            json={"question": "hello"}
        )
        
        assert response.status_code == 200
        data = response.json()
        assert "answer" in data
        assert len(data["answer"]) > 0  # Non-empty answer
    
    finally:
        # Cleanup
        proc.terminate()
        proc.wait(timeout=5)
```

---

**Test 2: Verify no Flask or stdio references**

**Given:** The updated `README.md`  
**When:** The file is read  
**Then:** It contains no Flask or stdio references

**Pseudocode:**
```python
def test_readme_no_flask_or_stdio():
    """
    Verify README does not contain Flask or stdio references.
    """
    with open("README.md", "r") as f:
        content = f.read().lower()
    
    assert "flask" not in content, "README still references Flask"
    assert "stdio" not in content, "README still references stdio"
    assert "subprocess" not in content, "README still references subprocess"
    assert "stdio_mcp" not in content, "README still references stdio_mcp"
```

---

**Test 3: Verify all four tools documented with ID normalization**

**Given:** The updated `README.md`  
**When:** The file is read  
**Then:** It documents all four MCP tools and includes get-by-id examples with ID normalization format matching Story 3's implementation

**Pseudocode:**
```python
def test_readme_documents_all_four_tools_and_id_format():
    """
    Verify README documents all four MCP tools per AC-2 and ID format per AC-5.
    
    Ensures:
    - All four tool names appear
    - Example questions cover get-by-id tools with ID extraction
    - ID normalization format is documented (matches Story 3's _normalize_id() implementation)
    
    NOTE: This test validates structure only. Actual ID format examples 
    (e.g., emp-001 → EMP-001 vs. emp001 → EMP001) MUST match Story 3's 
    investigation findings documented in mcp_client.py::_normalize_id().
    """
    with open("README.md", "r") as f:
        content = f.read()
    
    # Verify all four tool names appear in README
    assert "list_sales_order" in content, "README missing list_sales_order tool"
    assert "list_employees" in content, "README missing list_employees tool"
    assert "get_sales_order_by_id" in content, "README missing get_sales_order_by_id tool"
    assert "get_employee_by_id" in content, "README missing get_employee_by_id tool"
    
    # Verify example questions cover get-by-id tools with ID extraction
    assert "get employee" in content.lower(), "README missing get employee by ID example"
    assert "get sales order" in content.lower(), "README missing get sales order by ID example"
    
    # Verify ID normalization section exists (exact format validated against Story 3's impl)
    assert "id format" in content.lower(), "README missing ID Format section"
    assert "normalize" in content.lower() or "uppercase" in content.lower(), "README missing normalization description"
    
    # Check for PLACEHOLDER text (must be replaced with Story 3 findings)
    assert "PLACEHOLDER" not in content.upper(), "README still contains PLACEHOLDER text; Story 3 ID format investigation findings not yet incorporated"
    assert "IMPLEMENTATION INSTRUCTION" not in content.upper(), "README still contains IMPLEMENTATION INSTRUCTION; replace with Story 3 findings"
```

---

## Verification Summary

**Total test count:** 1 test (per breakdown Verification Approach: end-to-end README setup)

| Test | File | Coverage |
|---|---|---|
| Follow README setup from clean environment | `tests/test_readme_setup.py::test_readme_setup_end_to_end` | AC-5 (README primary path) |

**Additional validation tests (not counted toward breakdown's 1 test):**

| Test | File | Coverage |
|---|---|---|
| Verify no Flask or stdio references | `tests/test_readme_setup.py::test_readme_no_flask_or_stdio` | AC-5 (Flask/stdio removed) |
| Verify all four tools documented with ID format | `tests/test_readme_setup.py::test_readme_documents_all_four_tools_and_id_format` | AC-2 (all four tools), AC-5 (ID format documented, matches Story 3 findings) |

**Run tests:**

```bash
pytest tests/test_readme_setup.py -v
```

---

## Risks

### R-6: ID format documentation diverges from Story 3 implementation

**Likelihood:** Medium (if Story 4 written before Story 3 completes)  
**Impact:** High (README documents wrong behavior; user confusion; regression reports)  
**Mitigation:**
- **Story 4 MUST wait for Story 3 to complete** before finalizing README ID Format section.
- README template includes explicit `[PLACEHOLDER]` and `[IMPLEMENTATION INSTRUCTION]` tags that Dev replaces with Story 3's `_normalize_id()` docstring content.
- Test 3 (`test_readme_documents_all_four_tools_and_id_format`) fails if `PLACEHOLDER` or `IMPLEMENTATION INSTRUCTION` text remains, blocking merge until Story 3 findings are incorporated.
- If Story 4 must proceed in parallel with Story 3 (e.g., for review), leave placeholder text and file blocker comment: "Blocked on Story 3 ID normalization investigation findings."

**Alternative mitigation (if Story 4 cannot wait):**
- Document baseline assumption only (`prefix-digits` with hyphen, e.g., `emp-001` → `EMP-001`).
- Add prominent note: "ID normalization behavior is under investigation. Supported formats may expand in final release. See Story 3 for details."
- File follow-up ticket to update README after Story 3 completes.

---

## Notes

- **All four tools documented:** Usage section includes explicit examples for `list_sales_order`, `list_employees`, `get_employee_by_id`, and `get_sales_order_by_id`, satisfying AC-2.
- **ID normalization format documented:** README includes a dedicated "ID Format" subsection under Usage. **Dev MUST replace placeholder text with Story 3's `_normalize_id()` docstring content** to ensure parity with implementation. This satisfies AC-5.
- **Story dependency:** Story 4 **MUST NOT be merged until Story 3 completes** and its ID normalization investigation findings are documented in `mcp_client.py::_normalize_id()`. Dev replaces README placeholder with that content.
- **Example `MCP_URL` value:** `http://127.0.0.1:8000/sse` included in Configuration section.
- **LLM setup reference:** README documents both `ANTHROPIC_API_KEY` and `LLM_GATEWAY_URL` as alternatives; developer chooses based on their setup. If neither is set, agent returns `{"error": "LLM unavailable"}` (covered by Story 2 tests).
- **Flask and stdio removal:** All references removed; verified by `test_readme_no_flask_or_stdio`.
- **Closing story:** This story closes the epic when AC-5 is verified (README setup works end-to-end, no Flask/stdio, all four tools documented, ID format documented matching Story 3 implementation).
- **Development section:** Included in full README template above.
- **Existing README sections:** Preserve **License**, **Contributing**, **Changelog** sections if they exist and do not reference Flask/stdio. Append them at the end of the new README.
- **README template:** The content in Algorithm Notes is the full README; replaces existing `README.md` entirely unless preserving License/Contributing/Changelog sections.
- **Test count alignment:** Breakdown Verification Approach lists 1 test (end-to-end README setup). The two additional tests (`test_readme_no_flask_or_stdio`, `test_readme_documents_all_four_tools_and_id_format`) are **validation helpers** that run alongside the main test but do not change the breakdown's test count. They verify AC-5 (Flask/stdio removed, ID format documented) and AC-2 (all four tools) coverage without requiring a full end-to-end environment.
- **AC-3 fix:** While this story only updates README, the error handling fix from Stories 2 and 3 (returning tool errors directly without LLM synthesis) is now **documented** in the README's "Response (error)" example, showing `{"error": "MCP server unreachable"}` as a clear error return, consistent with AC-3.
- **ID format scope clarification (per fix-instruction.md):** The exception criterion "ID arguments are normalized the way they are today" requires parity with the original stdio MCP agent's behavior. **Story 3 is responsible for investigating what ID formats were historically supported** (e.g., `emp-001`, `emp001` without hyphen, multi-part prefixes) by examining original agent code, requirements, or tests. **Story 4 documents those findings** verbatim from Story 3's `_normalize_id()` docstring. This prevents the risk of silently breaking support for non-hyphenated IDs if the original agent handled them. The README template's placeholder text MUST be replaced with Story 3's actual investigation results before Story 4 can merge.
- **AC-2 FastMCP SSE transport verification deferred to Story 3:** This story (Story 4) only documents the FastAPI setup and usage in the README. **AC-2 (FastMCP SSE transport protocol) verification is the responsibility of Story 3**, which implements `agent/mcp_client.py` using the `mcp` library's `sse_client` and `ClientSession` for proper FastMCP SSE protocol compliance. Story 3's tests (e.g., `tests/test_tool_call.py`) mock the `mcp.client.sse.sse_client` and `mcp.ClientSession` to validate that the actual transport contract is correctly used. This story's end-to-end test (`test_readme_setup_end_to_end`) validates only that a developer can follow the README to run the agent and get a response; it does **not** re-verify the FastMCP SSE transport implementation, as that is already covered by Story 3's integration tests. This division of responsibility ensures Story 4 stays focused on documentation verification (AC-5) without duplicating Story 3's transport protocol tests.

---

## Open Questions

_None. README structure, content, removed references, all four tools documented, and verification tests are fully specified. ID normalization format documentation is **explicitly tied to Story 3's investigation findings** via placeholder replacement mechanism, ensuring parity with implementation and preventing silent regression of historically supported ID formats. AC-2 FastMCP SSE transport verification is explicitly the responsibility of Story 3's tests, not this story's end-to-end README test._
