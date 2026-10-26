# Course correction PER-5 (critic alignment)

Authoritative for this revision. Align requirements.md, hld.md, breakdown.md, and stories/01-graphql.md.

## Repo scope (unchanged)
- Only hub repo `graphql` (adlc-graphql).
- No mcp or agent work.

## Fix breakdown ↔ story ↔ HLD mismatches
Source of truth for implementation detail: **HLD + stories/01-graphql.md** (full GraphQL service).

1. In **breakdown.md Story 1 Allowed paths**, do NOT call `app/schema.py` a stub.
   - Describe it as the full schema: all four source types and all Query resolvers
     (salesorder_list, employee_list, get_salesorder_by_id, get_employee_by_id,
     plus HR Passport and Legal queries).

2. Add these to Story 1 Allowed paths (must match story File Plan + HLD file structure):
   - `app/mocks.py` (mock data for all sources)
   - `tests/test_graphql.py` (8 automated tests)

3. Keep Story 1 Verification Approach at **8 automated tests** (same as HLD 6.1 and story tests).

4. In **stories/01-graphql.md**:
   - Do not paste full epic ACs verbatim.
   - Reference breakdown/requirements ACs (AC1–AC3) briefly; keep File Plan, interfaces, and 8 tests.
   - File Plan must list: `app/__init__.py`, `app/main.py`, `app/schema.py` (full),
     `app/mocks.py`, `tests/test_graphql.py`.

5. Ensure **hld.md** file structure matches that same file list (no stub wording for schema.py).

## Do not change
- Ticket required operations and out-of-scope (no auth, no real integrations).
- Single-repo graphql scope.
