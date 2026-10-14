# breakdown.md

**Epic**: [graphql] GraphQL service: mock schemas & data for Salesforce, HR Prod, HR Passport, Legal  
**Date**: 2025-01-24  
**Status**: Draft

---

## 1. Overview

This breakdown slices the epic into a single implementation story that delivers the complete GraphQL HTTP endpoint with mock data for all four sources (Salesforce, HR Prod, HR Passport, Legal) and automated tests.

---

## 2. Acceptance Criteria (from requirements.md)

### AC1: Schemas and mock data exist for all four sources

**Given** the GraphQL service is initialized  
**When** schemas are defined  
**Then** types exist for Salesforce (SalesOrder), HR Prod (Employee), HR Passport, and Legal  
**And** mock data files or in-memory fixtures provide representative payloads for each type

---

### AC2: Required operations return sensible mock payloads

**Given** the GraphQL endpoint is running at POST /graphql  
**When** a client queries `salesorder_list`  
**Then** a list of mock SalesOrder objects is returned  

**When** a client queries `employee_list`  
**Then** a list of mock Employee objects is returned  

**When** a client queries `get_salesorder_by_id(id: ID!)`  
**Then** a single mock SalesOrder matching the requested ID is returned  

**When** a client queries `get_employee_by_id(id: ID!)`  
**Then** a single mock Employee matching the requested ID is returned  

**When** a client queries HR Passport or Legal data via their respective GraphQL types  
**Then** mock payloads are returned for those systems

---

### AC3: Basic automated tests cover the GraphQL operations

**Given** test cases are written for the GraphQL service  
**When** tests are executed  
**Then** all four required operations (`salesorder_list`, `employee_list`, `get_salesorder_by_id`, `get_employee_by_id`) pass with expected mock responses  
**And** at least one test covers HR Passport query behavior  
**And** at least one test covers Legal query behavior

---

## 3. Story Breakdown

### Story 1: GraphQL service with mock schemas, data, and tests for all four sources

**ID**: 1  
**Type**: story  
**Title**: GraphQL service with mock schemas, data, and tests for all four sources

**Description**:  
Implement complete GraphQL HTTP service using FastAPI and Strawberry. Define schemas for all four sources (Salesforce SalesOrder, HR Prod Employee, HR Passport, Legal), provide mock data fixtures, implement all required query resolvers, and deliver automated test suite covering all operations.

**Requirements coverage**:  
- AC1: Schemas and mock data for all four sources (SalesOrder, Employee, Passport, LegalCase types; mock data in `app/mocks.py`)
- AC2: All required operations return mock payloads (six Query resolvers: `salesorder_list`, `employee_list`, `get_salesorder_by_id`, `get_employee_by_id`, `passport_list`, `legalcase_list`)
- AC3: Automated tests verify all operations (8 test cases in `tests/test_graphql.py`)

**Allowed file paths**:  
- `app/__init__.py` (Python package marker)
- `app/main.py` (FastAPI app initialization; mounts `/graphql` endpoint)
- `app/schema.py` (full GraphQL schema: all four source types and all Query resolvers)
- `app/mocks.py` (mock data for all sources)
- `tests/test_graphql.py` (8 automated tests)

**Verification approach**:  
8 automated tests — see `stories/01-graphql.md` Tests section:
1. `test_salesorder_list` — returns list of ≥2 SalesOrder objects
2. `test_employee_list` — returns list of ≥2 Employee objects
3. `test_get_salesorder_by_id_valid` — returns single SalesOrder for valid ID
4. `test_get_salesorder_by_id_invalid` — returns null for invalid ID
5. `test_get_employee_by_id_valid` — returns single Employee for valid ID
6. `test_get_employee_by_id_invalid` — returns null for invalid ID
7. `test_passport_list` — returns list of ≥1 Passport objects
8. `test_legalcase_list` — returns list of ≥1 LegalCase objects

**Dependencies**: None (single story, all work contained within)

**Estimated complexity**: Medium (new service, but straightforward mock implementation)

---

## 4. Traceability

| AC | Story | Verification |
|----|-------|--------------|
| AC1 | Story 1 | Types defined in `app/schema.py`; mock data in `app/mocks.py` |
| AC2 | Story 1 | Six Query resolvers in `app/schema.py`; tests 1–8 verify responses |
| AC3 | Story 1 | 8 automated tests in `tests/test_graphql.py` |

---

## 5. Implementation Notes

- **Stack**: Python 3.9+, FastAPI, Strawberry GraphQL, pytest
- **Mock data storage**: In-memory Python dictionaries (per HLD AD-2)
- **GraphQL endpoint**: `POST /graphql` (per HLD AD-1)
- **ID lookup behavior**: First match or null (per HLD AD-4)
- **HR Passport and Legal**: Minimal but complete types and queries (per HLD AD-5)

---

## 6. Out of Scope

- Authentication (per ticket)
- Real Salesforce / HR / Legal integrations (per ticket)
- MCP server (per ticket)
- Agent code (per ticket)
- Pagination, caching, subscriptions (per HLD Future Considerations)

---

**End of breakdown**
