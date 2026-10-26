---
key: PER-6
type: story
status: todo
blocks: []
blocked_by: []
labels:
- Feature
- repo:mcp
repos:
- mcp
summary: '[mcp] MCP server: tools that fetch data from the GraphQL service'
---

## Goal

Create an MCP server whose source of truth is the GraphQL server only (HTTP calls to GraphQL). Agents will call these MCP tools later.

## Required tools

1. list_sales_order
2. list_employees
3. get_sales_order_by_id
4. get_employee_by_id

## Out of scope

* Authentication
* Re-implementing GraphQL schemas/mock data (that is [PER-5](https://linear.app/personalgs/issue/PER-5/graphql-graphql-service-mock-schemas-and-data-for-salesforce-hr-prod) / adlc-graphql)
* Agent orchestration UI/runtime beyond invoking these tools

## Acceptance

* MCP tools are implemented 
* Each tool queries the GraphQL server and returns a clear JSON result/error shape
* Automated tests with GraphQL client mocked or against local GraphQL
