---
key: PER-5
type: story
status: todo
blocks: []
blocked_by: []
labels:
- Feature
- repo:graphql
repos:
- graphql
summary: '[graphql] GraphQL service: mock schemas & data for Salesforce, HR Prod,
  HR Passport, Legal'
---

## 

## Goal

Create GraphQL schemas and representative mock data for:

* Salesforce
* HR Prod
* HR Passport
* Legal

Expose a single GraphQL HTTP API (e.g. POST /graphql) with consistent mock responses for testing before real source integrations.

## Required API operations

1. salesorder_list
2. employee_list
3. get_salesorder_by_id
4. get_employee_by_id

Also cover HR Passport and Legal via GraphQL queries/types in the same service (mock only).

## Out of scope

* Authentication
* Real Salesforce / HR / Legal integrations
* MCP server
* Agent code

## Acceptance

* Schemas + mock data live 
* All required operations return sensible mock payloads
* Basic automated tests for the GraphQL operations
