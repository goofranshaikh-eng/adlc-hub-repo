# Course correction PER-6 (critic / truncated artifacts)

Authoritative for this revision. Rewrite incomplete files fully — do not leave mid-sentence cuts.

## Repo scope
- Only hub repo `mcp` (adlc-mcp-server).
- GraphQL is an HTTP dependency (PER-5 service); do not implement GraphQL schemas here.
- No agent-repo work.

## Required tools (ticket)
1. list_sales_order
2. list_employees
3. get_sales_order_by_id
4. get_employee_by_id

Error shape for tools (from HLD): `{"error": "<message>"}` on failure.

## Fix truncated / incomplete artifacts (mandatory)
1. **breakdown.md** — currently truncated mid-title. Rewrite a **complete** breakdown.
   - Prefer **one implementable story** (`01-mcp`) that covers the whole MCP server
     (client + server + four tools + tests), OR fully specify every numbered story
     with ACs, allowed paths, and verification — no cutoffs.
   - Allowed paths must list every source and test file the story will create.
   - Verification test counts must match the story Tests section exactly.

2. **stories/01-mcp.md** — currently truncated mid-algorithm (`Catch ValueError:`).
   - Complete the MCP Server (`main`) algorithm end-to-end.
   - Include a full **Tests** section (not only mentioned in ACs).
   - Finish all sections; no mid-sentence endings.
   - If output would be long: keep prose tight, but every section must be complete.

3. Align **requirements.md** AC counts with story/breakdown:
   - Keep 3 top-level ACs (AC1 tools, AC2 behaviors/errors, AC3 tests) OR
     renumber consistently everywhere.
   - Do not invent extra Given/When/Then scenarios in the story that are absent
     from requirements/breakdown — either add them to requirements+breakdown
     or remove them from the story.
   - Test count: pick one agreed number (e.g. success + error coverage for all
     four tools) and use that same count in requirements AC3, breakdown
     verification, and story Tests.

4. **hld.md** — keep contracts (tool names, JSON shapes, GRAPHQL_ENDPOINT).
   Ensure file structure matches the completed story File Plan.

## Out of scope
- Authentication
- Implementing GraphQL mock schemas (PER-5)
- Agent orchestration (PER-7)

## Done when
Critic can PASS: no truncated files, AC/test counts match across
requirements ↔ breakdown ↔ stories/01-mcp.md ↔ HLD.
