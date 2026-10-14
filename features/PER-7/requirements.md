# Requirements: Agent Query MCP Tools for Sales Orders and Employees

**Epic:** [agent] Agent: query MCP tools for sales orders and employees  
**Document Version:** 4.0  
**Last Updated:** 2025-01-20  
**Status:** Draft

---

## Overview

This document defines requirements for an agent that queries MCP tools for sales orders and employees. The agent translates natural language queries into appropriate MCP tool invocations and formats responses for users. This epic delivers as a **single story** (`01-agent`) covering MCP client wrapper, agent core logic, and verification.

---

## Scope

### In Scope
- Agent code that invokes four MCP tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`)
- Natural language query processing (keyword-based intent parsing)
- Response formatting (human-readable answers)
- Basic automated tests verifying all four tools invocable

### Out of Scope
- Authentication (assumed handled by MCP tools)
- Implementing GraphQL schemas (dependency PER-5)
- Implementing MCP server tools (dependency PER-6)

---

## Acceptance Criteria

### AC1: Agent code exists
**Given** the agent implementation is complete  
**When** the agent codebase is reviewed  
**Then** agent code exists in the repository

**Verification:**
- Agent source files exist: `agent/src/mcp_client.py`, `agent/src/agent_core.py`
- Test files exist: `agent/tests/test_mcp_client.py`, `agent/tests/test_agent.py`

---

### AC2: Agent can invoke all four MCP tools and return useful answers
**Given** the agent has access to the four MCP tools from PER-6  
**When** a user queries about sales orders or employees  
**Then** the agent invokes the appropriate MCP tool(s) and returns a useful answer

**Verification:**
- Query `"list sales orders"` → agent invokes `list_sales_order`, returns formatted list (e.g., `"Found 3 sales orders: SO-1, SO-2, SO-3"`)
- Query `"list employees"` → agent invokes `list_employees`, returns formatted list (e.g., `"Found 2 employees: Alice, Bob"`)
- Query `"get sales order SO-1"` → agent invokes `get_sales_order_by_id`, returns formatted details (e.g., `"Sales Order SO-1: Amount: 1000"`)
- Query `"get employee E-1"` → agent invokes `get_employee_by_id`, returns formatted details (e.g., `"Employee Alice (ID: E-1): Department: Engineering"`)

---

### AC3: Basic tests or documented manual verification script
**Given** the agent implementation is complete  
**When** verification is required  
**Then** either automated tests exist OR a documented manual verification script is provided

**Verification:**
- 8 automated tests exist (4 MCP client unit tests + 4 agent integration tests)
- All tests pass when MCP server (PER-6) is accessible

---

## Requirements

### REQ-1: MCP Tool Invocation (Maps to AC1, AC2)
**Description:** Agent must invoke all four MCP tools via JSON-RPC over HTTP  
**Priority:** High  
**Rationale:** Core functionality enabling agent to fetch sales order and employee data

**Details:**
- MCP client wrapper provides four methods: `invoke_list_sales_order()`, `invoke_list_employees()`, `invoke_get_sales_order_by_id(order_id)`, `invoke_get_employee_by_id(employee_id)`
- Connection via `MCP_SERVER_URL` environment variable
- JSON-RPC 2.0 format with `tools/call` method
- Success returns dictionary with `data` key; errors return `None`

**Verification:**
- 4 unit tests verify each MCP client method invokes correct tool with correct parameters

---

### REQ-2: Query Processing (Maps to AC2)
**Description:** Agent must parse natural language queries and route to appropriate MCP tools  
**Priority:** High  
**Rationale:** Enables users to interact with agent using natural language

**Details:**
- Supported query patterns:
  - `"list sales orders"` → `list_sales_order`
  - `"list employees"` → `list_employees`
  - `"get sales order <id>"` → `get_sales_order_by_id`
  - `"get employee <id>"` → `get_employee_by_id`
- Keyword matching (not LLM-based)
- ID extraction via regex: `\b([a-z0-9\-]+)\s*$`

**Verification:**
- 4 integration tests verify query processing for all four query patterns

---

### REQ-3: Response Formatting (Maps to AC2)
**Description:** Agent must format tool responses into human-readable answers  
**Priority:** High  
**Rationale:** AC2 requires "useful answers" (implies human-readable)

**Details:**
- List queries: output format `"Found N items: [summary]"` (first 5 IDs/names, then count remainder)
- Get-by-id queries: output format `"<Entity> <name> (ID: <id>): [key details]"` (amount for sales orders, department for employees)
- Errors: output format `"Error: <message>"`

**Verification:**
- 4 integration tests verify response formats match specifications

---

### REQ-4: Error Handling (Maps to AC2)
**Description:** Agent must handle errors gracefully and return informative messages  
**Priority:** Medium  
**Rationale:** Ensures agent remains usable when tools unavailable or queries malformed

**Details:**
- Network errors: MCP client logs warning, returns `None`; agent returns `"Error: Unable to fetch <entities>"`
- HTTP errors (4xx, 5xx): same as network errors
- Tool errors (ID not found): agent returns `"Error: <Entity> <id> not found"`
- Unknown query intent: agent returns `"Error: Unable to understand query. Try 'list sales orders', 'list employees', 'get sales order <id>', or 'get employee <id>'"`
- Missing ID in get-by-id query: agent returns `"Error: No <entity> ID provided. Use 'get <entity> <id>'"`
- Empty list response: agent returns `"No <entities> found"`

**Verification:**
- Integration tests verify error messages for tool returns `None` scenarios

---

### REQ-5: Test Coverage (Maps to AC3)
**Description:** Automated tests must verify all four MCP tools invocable and responses formatted correctly  
**Priority:** High  
**Rationale:** AC3 requires "basic tests"

**Details:**
- 4 MCP client unit tests (one per tool method success path)
- 4 agent integration tests (one per query pattern, end-to-end verification)
- Total: 8 tests

**Verification:**
- Test suite runs successfully; all 8 tests pass

---

## Story

### Story 1: Agent: query MCP tools for sales orders and employees

**Description:**  
Build a complete agent that queries MCP tools for sales orders and employees. Includes MCP client wrapper (four tool methods), agent core query processing (keyword-based intent parsing and response formatting), and verification tests. Delivers all three acceptance criteria (AC1, AC2, AC3) in a single implementation slice.

**Covers:**  
- REQ-1 (MCP tool invocation)
- REQ-2 (query processing)
- REQ-3 (response formatting)
- REQ-4 (error handling)
- REQ-5 (test coverage)

**Acceptance Criteria:**
- AC1: Agent code exists in the repository
- AC2: Agent can invoke all four MCP tools and return useful answers
- AC3: Basic tests or documented manual verification script

**Files:**
- `agent/src/mcp_client.py` (new)
- `agent/src/agent_core.py` (new)
- `agent/tests/test_mcp_client.py` (new)
- `agent/tests/test_agent.py` (new)

**Verification:**
- 8 tests total (4 MCP client unit tests + 4 agent integration tests)

---

## Dependencies

### External
- **PER-6 (MCP Server Tools):** Must be complete and accessible before agent can invoke tools
- **PER-5 (GraphQL Schemas):** Indirect dependency (tools use schemas, agent does not)
- **Python `requests` library:** HTTP client for JSON-RPC calls
- **Python `pytest` library:** Test framework

### Internal
- None (single story delivers all components)

---

## Technical Constraints

1. **Connection:** MCP client connects via `MCP_SERVER_URL` environment variable (or constructor parameter override)
2. **Tool Protocol:** JSON-RPC 2.0 format with `tools/call` method
3. **Query Parsing:** Keyword matching (not LLM-based); deterministic and sufficient for four supported patterns
4. **Response Format:** Plain text strings (not structured data); human-readable
5. **Error Handling:** All runtime errors return falsy values (`None`, error message strings), never raise exceptions after initialization
6. **Repository:** All files under `agent/` subtree
7. **Stack:** Python

---

## Non-Functional Requirements

### Performance
- MCP tool invocations timeout after 10 seconds
- Query processing latency determined primarily by MCP tool response times

### Reliability
- Agent handles network errors, HTTP errors, and malformed responses gracefully
- All errors logged at WARNING level with tool name and error details

### Maintainability
- MCP client isolates tool interface changes (update one method per tool when tool signatures change)
- Agent core separates query parsing from response formatting

---

## Coverage Mapping

| AC | Requirements | Story | Verification Tests |
|----|--------------|-------|-------------------|
| AC1: Agent code exists | REQ-1, REQ-5 | Story 1 | File existence check (4 source files exist) |
| AC2: Agent can invoke all four MCP tools and return useful answers | REQ-1, REQ-2, REQ-3, REQ-4 | Story 1 | 4 integration tests confirm all four tools invocable and responses formatted correctly |
| AC3: Basic tests or documented manual verification script | REQ-5 | Story 1 | 8 automated tests (4 unit + 4 integration) satisfy "basic tests" requirement |

**Total Verification Tests:** 8 (4 MCP client unit tests + 4 agent integration tests)

---

## Open Questions

1. ~~**MCP tool connection details:** How does the agent connect to MCP tools?~~  
   **Resolved:** Environment variable `MCP_SERVER_URL`

2. ~~**Query pattern examples:** What exact query strings should be supported?~~  
   **Resolved:** Minimal set: `"list sales orders"`, `"list employees"`, `"get sales order <id>"`, `"get employee <id>"`

3. ~~**Response detail level:** Should list queries return full details or just IDs/names?~~  
   **Resolved:** Summary (count + first 5 IDs/names) for lists, full key details for get-by-id

---

## Notes

- **Single-story implementation:** All three ACs delivered in one story (`01-agent`)
- Authentication out of scope (assumed handled by MCP tools or infrastructure)
- Agent does not implement GraphQL or MCP tools (dependencies from PER-5/PER-6)
- Response formatting prioritizes readability over structured data
- Test count is **8** (4 MCP client unit tests + 4 agent integration tests), aligned across all artifacts
- All file paths are under `agent/` subtree (matching repo-tree context); stack is Python (matching project-context)
- No invented acceptance criteria beyond ticket AC1, AC2, AC3
- Error handling follows fail-safe pattern: log and return falsy values, never crash

---

**End of Requirements**
