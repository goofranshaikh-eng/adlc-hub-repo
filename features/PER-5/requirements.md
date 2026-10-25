# stories/01-graphql.md

**Repo**: graphql  
**Type**: story  
**Status**: Ready for implementation

---

## 1. Acceptance Criteria

References breakdown/requirements AC1–AC3:

- **AC1**: Schemas and mock data exist for all four sources (SalesOrder, Employee, Passport, LegalCase types with mock data fixtures)
- **AC2**: All required operations return sensible mock payloads (six Query resolvers functional)
- **AC3**: Basic automated tests cover all GraphQL operations (8 test cases pass)

---

## 2. File Plan

### 2.1 New Files

| Path | Purpose |
|------|---------|
| `app/__init__.py` | Python package marker (empty) |
| `app/main.py` | FastAPI application; mounts `/graphql` endpoint via Strawberry GraphQLRouter |
| `app/schema.py` | Strawberry type definitions (SalesOrder, Employee, Passport, LegalCase) and Query class with all six resolvers (salesorder_list, employee_list, get_salesorder_by_id, get_employee_by_id, passport_list, legalcase_list) |
| `app/mocks.py` | Mock data dictionaries for all four sources (SALESFORCE_ORDERS, HR_EMPLOYEES, HR_PASSPORTS, LEGAL_CASES) |
| `tests/test_graphql.py` | Automated pytest test suite for all GraphQL operations (8 tests) |

### 2.2 Modified Files

None (repository is empty per repo-tree).

---

## 3. Implementation Interfaces

### 3.1 app/main.py

**Purpose**: Initialize FastAPI application and mount GraphQL endpoint.

**Interface**:
```python
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from app.schema import schema

app: FastAPI
graphql_app: GraphQLRouter
```

**Key behaviors**:
- Create FastAPI instance
- Create Strawberry GraphQLRouter with schema from `app.schema`
- Mount router at `/graphql` path
- Expose `app` for uvicorn to run

---

### 3.2 app/schema.py

**Purpose**: Define complete GraphQL schema (all four source types and Query resolvers).

**Interface**:
```python
import strawberry
from typing import Optional

@strawberry.type
class SalesOrder:
    id: str
    account_name: str
    amount: float
    status: str

@strawberry.type
class Employee:
    id: str
    first_name: str
    last_name: str
    department: str
    email: str

@strawberry.type
class Passport:
    id: str
    employee_id: str
    issue_date: str
    expiry_date: str

@strawberry.type
class LegalCase:
    id: str
    case_number: str
    summary: str
    status: str

@strawberry.type
class Query:
    @strawberry.field
    def salesorder_list(self) -> list[SalesOrder]: ...
    
    @strawberry.field
    def employee_list(self) -> list[Employee]: ...
    
    @strawberry.field
    def get_salesorder_by_id(self, id: str) -> Optional[SalesOrder]: ...
    
    @strawberry.field
    def get_employee_by_id(self, id: str) -> Optional[Employee]: ...
    
    @strawberry.field
    def passport_list(self) -> list[Passport]: ...
    
    @strawberry.field
    def legalcase_list(self) -> list[LegalCase]: ...

schema = strawberry.Schema(query=Query)
```

**Key behaviors**:
- `salesorder_list`: Returns all SalesOrder objects from `mocks.SALESFORCE_ORDERS`
- `employee_list`: Returns all Employee objects from `mocks.HR_EMPLOYEES`
- `get_salesorder_by_id(id)`: Searches `mocks.SALESFORCE_ORDERS` for matching `id`, returns first match or `None`
- `get_employee_by_id(id)`: Searches `mocks.HR_EMPLOYEES` for matching `id`, returns first match or `None`
- `passport_list`: Returns all Passport objects from `mocks.HR_PASSPORTS`
- `legalcase_list`: Returns all LegalCase objects from `mocks.LEGAL_CASES`

**Notes**:
- Field names use snake_case per Python conventions; GraphQL exposes as camelCase automatically via Strawberry
- ID lookups use simple iteration and string equality (per HLD AD-4: "first match or None")

---

### 3.3 app/mocks.py

**Purpose**: Provide in-memory mock data for all four sources.

**Interface**:
```python
SALESFORCE_ORDERS: list[dict]
HR_EMPLOYEES: list[dict]
HR_PASSPORTS: list[dict]
LEGAL_CASES: list[dict]
```

**Mock data requirements**:

**SALESFORCE_ORDERS** (≥2 records):
```python
[
    {
        "id": "SO-001",
        "account_name": "Acme Corp",
        "amount": 15000.0,
        "status": "Open"
    },
    {
        "id": "SO-002",
        "account_name": "Globex Industries",
        "amount": 28500.0,
        "status": "Closed"
    }
]
```

**HR_EMPLOYEES** (≥2 records):
```python
[
    {
        "id": "EMP-001",
        "first_name": "Alice",
        "last_name": "Smith",
        "department": "Engineering",
        "email": "alice.smith@example.com"
    },
    {
        "id": "EMP-002",
        "first_name": "Bob",
        "last_name": "Jones",
        "department": "Sales",
        "email": "bob.jones@example.com"
    }
]
```

**HR_PASSPORTS** (≥1 record):
```python
[
    {
        "id": "PP-001",
        "employee_id": "EMP-001",
        "issue_date": "2023-01-15",
        "expiry_date": "2028-01-15"
    }
]
```

**LEGAL_CASES** (≥1 record):
```python
[
    {
        "id": "LC-001",
        "case_number": "2024-CV-00123",
        "summary": "Contract dispute with vendor",
        "status": "Active"
    }
]
```

---

### 3.4 tests/test_graphql.py

**Purpose**: Automated pytest test suite for all GraphQL operations.

**Interface**:
```python
from fastapi.testclient import TestClient
import pytest

@pytest.fixture
def client() -> TestClient: ...

def test_salesorder_list(client): ...
def test_employee_list(client): ...
def test_get_salesorder_by_id_valid(client): ...
def test_get_salesorder_by_id_invalid(client): ...
def test_get_employee_by_id_valid(client): ...
def test_get_employee_by_id_invalid(client): ...
def test_passport_list(client): ...
def test_legalcase_list(client): ...
```

**Key behaviors** (per HLD testing strategy):

1. **test_salesorder_list**: POST query `{ salesorder_list { id accountName amount status } }`, assert response data contains list of ≥2 SalesOrder objects with all fields populated
2. **test_employee_list**: POST query `{ employee_list { id firstName lastName department email } }`, assert response data contains list of ≥2 Employee objects with all fields populated
3. **test_get_salesorder_by_id_valid**: POST query `{ get_salesorder_by_id(id: "SO-001") { id accountName } }`, assert response data contains single SalesOrder with `id: "SO-001"`
4. **test_get_salesorder_by_id_invalid**: POST query `{ get_salesorder_by_id(id: "SO-999") { id } }`, assert response data contains `get_salesorder_by_id: null`
5. **test_get_employee_by_id_valid**: POST query `{ get_employee_by_id(id: "EMP-001") { id firstName } }`, assert response data contains single Employee with `id: "EMP-001"`
6. **test_get_employee_by_id_invalid**: POST query `{ get_employee_by_id(id: "EMP-999") { id } }`, assert response data contains `get_employee_by_id: null`
7. **test_passport_list**: POST query `{ passport_list { id employeeId issueDate expiryDate } }`, assert response data contains list of ≥1 Passport objects with all fields populated
8. **test_legalcase_list**: POST query `{ legalcase_list { id caseNumber summary status } }`, assert response data contains list of ≥1 LegalCase objects with all fields populated

**Notes**:
- Use `TestClient` from `fastapi.testclient` (wraps `httpx`)
- All queries POST to `/graphql` with JSON body `{"query": "..."}`
- Assert on `response.json()["data"]` structure and values
- No authentication headers required (out of scope per ticket)

---

## 4. Algorithm Notes

### 4.1 ID-based lookup resolver pattern

**Used by**: `get_salesorder_by_id`, `get_employee_by_id`

**Algorithm**:
```python
def get_salesorder_by_id(self, id: str) -> Optional[SalesOrder]:
    # Import mock data
    from app.mocks import SALESFORCE_ORDERS
    
    # Linear search for matching ID
    for order_dict in SALESFORCE_ORDERS:
        if order_dict["id"] == id:
            # Convert dict to Strawberry type
            return SalesOrder(**order_dict)
    
    # No match found
    return None
```

**Key points**:
- Simple linear search acceptable for mock data (small dataset)
- String equality comparison on `id` field
- Return first match only (per HLD AD-4)
- Return `None` (GraphQL nullable) when ID not found, not an error
- Dict-to-Strawberry-type conversion via `**kwargs` unpacking

---

### 4.2 List resolver pattern

**Used by**: `salesorder_list`, `employee_list`, `passport_list`, `legalcase_list`

**Algorithm**:
```python
def salesorder_list(self) -> list[SalesOrder]:
    # Import mock data
    from app.mocks import SALESFORCE_ORDERS
    
    # Convert all dicts to Strawberry types
    return [SalesOrder(**order_dict) for order_dict in SALESFORCE_ORDERS]
```

**Key points**:
- Return all records from corresponding mock data list
- List comprehension converts dicts to Strawberry types
- No filtering or pagination (out of scope)

---

### 4.3 FastAPI + Strawberry integration

**In app/main.py**:
```python
from fastapi import FastAPI
from strawberry.fastapi import GraphQLRouter
from app.schema import schema

app = FastAPI()
graphql_app = GraphQLRouter(schema)
app.include_router(graphql_app, prefix="/graphql")
```

**Key points**:
- `GraphQLRouter` handles GraphQL protocol (query parsing, execution, response formatting)
- Mount at `/graphql` prefix makes endpoint available at `POST /graphql`
- Strawberry automatically exposes GraphiQL playground at `GET /graphql` (dev convenience)
- No custom middleware or error handling needed for mock service

---

## 5. Tests

**Test count**: 8 automated tests (matches breakdown Verification Approach and HLD 6.1)

| Test | Verifies | AC Coverage |
|------|----------|-------------|
| `test_salesorder_list` | Returns list of ≥2 SalesOrder objects with all fields | AC2 (salesorder_list) |
| `test_employee_list` | Returns list of ≥2 Employee objects with all fields | AC2 (employee_list) |
| `test_get_salesorder_by_id_valid` | Returns single SalesOrder for valid ID | AC2 (get_salesorder_by_id) |
| `test_get_salesorder_by_id_invalid` | Returns null for invalid ID | AC2 (get_salesorder_by_id edge case) |
| `test_get_employee_by_id_valid` | Returns single Employee for valid ID | AC2 (get_employee_by_id) |
| `test_get_employee_by_id_invalid` | Returns null for invalid ID | AC2 (get_employee_by_id edge case) |
| `test_passport_list` | Returns list of ≥1 Passport objects with all fields | AC2 (HR Passport query) |
| `test_legalcase_list` | Returns list of ≥1 LegalCase objects with all fields | AC2 (Legal query) |

**Test Details**:

### Test 1: test_salesorder_list
**Verifies**: AC2 — `salesorder_list` returns list of mock SalesOrder objects  
**Given**: GraphQL endpoint is running  
**When**: Client POSTs query `{ salesorder_list { id accountName amount status } }` to `/graphql`  
**Then**: Response `data.salesorder_list` is a list containing ≥2 objects  
**And**: Each object has `id`, `accountName`, `amount`, `status` fields populated  
**And**: At least one object has `id: "SO-001"` and at least one has `id: "SO-002"`

---

### Test 2: test_employee_list
**Verifies**: AC2 — `employee_list` returns list of mock Employee objects  
**Given**: GraphQL endpoint is running  
**When**: Client POSTs query `{ employee_list { id firstName lastName department email } }` to `/graphql`  
**Then**: Response `data.employee_list` is a list containing ≥2 objects  
**And**: Each object has `id`, `firstName`, `lastName`, `department`, `email` fields populated  
**And**: At least one object has `id: "EMP-001"` and at least one has `id: "EMP-002"`

---

### Test 3: test_get_salesorder_by_id_valid
**Verifies**: AC2 — `get_salesorder_by_id` returns single SalesOrder for valid ID  
**Given**: GraphQL endpoint is running  
**When**: Client POSTs query `{ get_salesorder_by_id(id: "SO-001") { id accountName amount status } }` to `/graphql`  
**Then**: Response `data.get_salesorder_by_id` is a single object (not null)  
**And**: Object has `id: "SO-001"`  
**And**: Object has `accountName`, `amount`, `status` fields populated

---

### Test 4: test_get_salesorder_by_id_invalid
**Verifies**: AC2 — `get_salesorder_by_id` returns null for invalid ID (edge case clarification)  
**Given**: GraphQL endpoint is running  
**When**: Client POSTs query `{ get_salesorder_by_id(id: "SO-999") { id } }` to `/graphql`  
**Then**: Response `data.get_salesorder_by_id` is `null`

---

### Test 5: test_get_employee_by_id_valid
**Verifies**: AC2 — `get_employee_by_id` returns single Employee for valid ID  
**Given**: GraphQL endpoint is running  
**When**: Client POSTs query `{ get_employee_by_id(id: "EMP-001") { id firstName lastName department email } }` to `/graphql`  
**Then**: Response `data.get_employee_by_id` is a single object (not null)  
**And**: Object has `id: "EMP-001"`  
**And**: Object has `firstName`, `lastName`, `department`, `email` fields populated

---

### Test 6: test_get_employee_by_id_invalid
**Verifies**: AC2 — `get_employee_by_id` returns null for invalid ID (edge case clarification)  
**Given**: GraphQL endpoint is running  
**When
