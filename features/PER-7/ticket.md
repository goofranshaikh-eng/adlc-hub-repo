---
key: PER-7
type: story
status: todo
blocks: []
blocked_by: []
labels:
- Feature
- repo:agent
repos:
- agent
summary: '[agent] Agent: query MCP tools for sales orders and employees'
---

## Goal

Build/configure an agent that uses the MCP tools from [PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service) to answer queries about sales orders and employees.

## Must use MCP tools

1. list_sales_order
2. list_employees
3. get_sales_order_by_id
4. get_employee_by_id

## Out of scope

* Authentication
* Implementing GraphQL schemas ([PER-5](https://linear.app/personalgs/issue/PER-5/graphql-graphql-service-mock-schemas-and-data-for-salesforce-hr-prod))
* Implementing MCP server tools ([PER-6](https://linear.app/personalgs/issue/PER-6/mcp-mcp-server-tools-that-fetch-data-from-the-graphql-service))

## Acceptance

* Agent code 
* Agent can invoke all four MCP tools and return useful answers
* Basic tests or a documented manual verification script
