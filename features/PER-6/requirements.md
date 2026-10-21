# Requirements: MCP server tools that fetch data from the GraphQL service

**Ticket:** [mcp] MCP server: tools that fetch data from the GraphQL service

## Overview

Create an MCP (Model Context Protocol) server that exposes four tools for querying a GraphQL service over HTTP. The server's sole data source is the GraphQL endpoint; no local schemas or mock data are managed by this server.

## Acceptance Criteria

### AC1: MCP tools are implemented

**Given** the MCP server is running  
**When** an agent queries the available tools  
**Then** the server exposes exactly four tools: `list_sales_order`, `list_employees`, `get_sales_order_by_id`, `get_employee_by_id`

---

**Given** the MCP server is running  
**When** an agent invokes any of the four tools  
**Then** the server accepts the request and attempts to execute it

### AC2: Each tool queries the GraphQL server and returns a clear JSON result/error shape

**Given** `list_sales_order` is invoked  
**When** the GraphQL server returns sales order data  
**Then** the tool returns a JSON object with key `salesOrders` containing an array of sales orders

---

**Given** `list_employees` is invoked  
**When** the GraphQL server returns employee data  
**Then** the tool returns a JSON object with key `employees` containing an array of employees

---

**Given** `get_sales_order_by_id` is invoked with a valid ID  
**When** the GraphQL server returns the matching sales order  
**Then** the tool returns a JSON object with key `salesOrder` containing that sales order object

---

**Given** `get_sales_order_by_id` is invoked without an ID parameter  
**When** the tool validates input  
**Then** the tool returns `{"error": "Missing required parameter: id"}`

---

**Given** `get_sales_order_by_id` is invoked with an ID not found in the GraphQL service  
**When** the GraphQL server returns no matching sales order  
**Then** the tool returns `{"salesOrder": null}`

---

**Given** `get_employee_by_id` is invoked with a valid ID  
**When** the GraphQL server returns the matching employee  
**Then** the tool returns a JSON object with key `employee` containing that employee object

---

**Given** `get_employee_by_id` is invoked without an ID parameter  
**When** the tool validates input  
**Then** the tool returns `{"error": "Missing required parameter: id"}`

---

**Given** `get_employee_by_id` is invoked with an ID not found in the GraphQL service  
**When** the GraphQL server returns no matching employee  
**Then** the tool returns `{"employee": null}`

---

**Given** any tool is invoked  
**When** the GraphQL server returns an error or is unreachable  
**Then** the tool returns a JSON object with key `error` containing a clear error message

### AC3: Automated tests with GraphQL client mocked or against local GraphQL

**Given** the test suite is executed  
**When** tests run against a mocked GraphQL client  
**Then** all four tools are verified to construct correct GraphQL queries and return expected JSON shapes for success cases

---

**Given** the test suite is executed  
**When** tests simulate GraphQL errors (network failure, GraphQL error response, missing data)  
**Then** all four tools are verified to return clear JSON error objects with key `error`

---

**Given** integration tests are executed  
**When** tests run against a local GraphQL server  
**Then** all four tools are verified to query the live GraphQL endpoint and return the expected JSON shapes

## Out of Scope

- Authentication mechanisms
- Re-implementing GraphQL schemas or mock data (covered by PER-5 / adlc-graphql)
- Agent orchestration UI or runtime beyond tool invocation

## Open Questions

None.

## Notes

- The GraphQL server endpoint must be configurable via the `GRAPHQL_ENDPOINT` environment variable
- Error shapes must be consistent across all four tools: `{"error": "<message>"}`
- MCP protocol compliance is assumed; refer to MCP specification for tool registration and invocation contracts
- Test count: 18 automated tests total (4 GraphQL client unit tests + 12 tool unit tests + 2 integration tests)
