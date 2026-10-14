# Breakdown: Agent Query MCP Tools for Sales Orders and Employees

**Epic:** [agent] Agent: query MCP tools for sales orders and employees  
**Document Version:** 4.0  
**Last Updated:** 2025-01-20  
**Status:** Draft

---

## Overview

This breakdown delivers the agent epic as a **single implementation story** (`01-agent`) covering MCP client wrapper, agent core query processing, and verification. The story targets the `agent/` subtree. All three acceptance criteria (AC1, AC2, AC3) are delivered in one testable slice that reaches Done when all verification tests pass.

---

## Story Sequence

| ID | Title | Type | Prerequisites | Verification Tests |
|----|-------|------|---------------|-------------------|
| 01-agent | Agent: query MCP tools for sales orders and employees | story | — | 8 |

---

## Story 01-agent: Agent: query MCP tools for sales orders and employees

**Description:**  
Build a complete agent that queries MCP tools for sales orders and employees. Includes MCP client wrapper (four tool methods), agent core query processing (keyword-based intent parsing and response formatting), and verification tests. Delivers all three acceptance criteria (AC1, AC2, AC3) in a single implementation slice.

**Covers:**  
- REQ-1 (MCP tool invocation)
- REQ-2 (query processing)
- REQ-3 (response formatting)
- REQ-4 (error handling)
- REQ-5 (test coverage)

**Acceptance Criteria:**

1. **AC1:** Agent code exists in the repository
2. **AC2:** Agent can invoke all four MCP tools and return useful answers
3. **AC3:** Basic tests or documented manual verification script

**Verification Approach:**

- **MCP Client Unit Tests:** 4 tests (one per tool method success path)
  - `test_invoke_list_sales_order_success`
  - `test_invoke_list_employees_success`
  - `test_invoke_get_sales_order_by_id_success`
  - `test_invoke_get_employee_by_id_success`
- **Agent Integration Tests:** 4 tests (one per tool, verifying end-to-end flow)
  - `test_verify_list_sales_order`
  - `test_verify_list_employees`
  - `test_verify_get_sales_order_by_id`
  - `test_verify_get_employee_by_id`

**Test Strategy:**
- MCP client tests use mocked HTTP responses to verify tool invocation success paths
- Agent integration tests use mocked MCP client responses to verify query processing and response formatting
- All four MCP tools (`list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`) verified invocable with non-error responses

**Test count:** 8 (4 MCP client unit tests + 4 agent integration tests)

**Allowed Paths:**

- `agent/src/mcp_client.py` (new)
- `agent/src/agent_core.py` (new)
- `agent/tests/test_mcp_client.py` (new)
- `agent/tests/test_agent.py` (new)

**Prerequisites:**  
— (opens and closes the epic)

**Notes:**

- Connects to MCP tools via environment variable `MCP_SERVER_URL` (per HLD AD-4)
- Query parsing uses keyword matching (not LLM per HLD AD-1)
- Response formatting per HLD AD-3: readable text (`"Found N items: ..."` for lists, `"Entity <name> (ID: <id>): [details]"` for get-by-id)
- No caching or retry logic (thin wrapper per HLD AD-2)
- Error handling returns error messages; never raises exceptions after initialization (per HLD AD-5)
- See story LLD (`stories/01-agent.md`) for detailed test specifications and implementation guidance

---

## Dependencies

**External:**
- PER-6 (MCP Server Tools) must be complete and accessible before this story can invoke tools
- PER-5 (GraphQL Schemas) is an indirect dependency (tools use schemas, agent does not)
- Python `requests` library (HTTP client for JSON-RPC calls)
- Python `pytest` library (test framework)

**Inter-story:**
- None (single story delivers entire epic)

---

## Risks

| Risk | Story | Mitigation |
|------|-------|------------|
| MCP tool signatures change (PER-6) | 01-agent | Isolate changes in MCP client; update one method per tool |
| Query parsing fails on unexpected input | 01-agent | Document supported patterns; return helpful error |
| MCP server unavailable at runtime | 01-agent | MCP client logs errors and returns `None`; agent returns error message |

---

## Acceptance Criteria Summary

| AC | Story Coverage | Verification Files | Test Count |
|----|----------------|-------------------|------------|
| AC1: Agent code exists | 01-agent | `agent/src/mcp_client.py`, `agent/src/agent_core.py`, `agent/tests/test_mcp_client.py`, `agent/tests/test_agent.py` | File existence check (4 source files exist) |
| AC2: Agent can invoke all four MCP tools and return useful answers | 01-agent | `agent/src/agent_core.py`, `agent/tests/test_agent.py` | 4 integration tests (verify all four tools invocable and responses formatted correctly per HLD AD-3) |
| AC3: Basic tests or documented manual verification script | 01-agent | `agent/tests/test_mcp_client.py`, `agent/tests/test_agent.py` | 8 automated tests (4 MCP client unit tests + 4 agent integration tests satisfy "basic tests" requirement) |

---

## Notes

- **Single story delivers all three ACs:** AC1 (agent code exists), AC2 (invokes all four tools and returns useful answers), AC3 (basic tests)
- All new files under `agent/`; no existing repo files modified
- Authentication out of scope (assumed handled by MCP tools)
- MCP client connection via `MCP_SERVER_URL` environment variable (HLD open question resolved)
- **Total verification tests:** 8 (4 MCP client unit tests + 4 agent integration tests)
- **Test count alignment:**
  - Ticket AC3: "Basic tests or a documented manual verification script"
  - Requirements.md Coverage Mapping: 8 total tests (4 unit + 4 integration)
  - HLD AC coverage section: 8 tests
  - Story LLD Tests section: 8 tests listed
  - This breakdown: 8 tests
  - **Alignment:** ✅ Test count matches ticket, requirements, HLD, story LLD, and breakdown exactly
- All file paths are under `agent/` subtree (matching repo-tree context); stack is Python (matching project-context)
- No invented acceptance criteria beyond ticket AC1, AC2, AC3

---

**End of Breakdown**
