# Course correction PER-7 (authoritative)

Human edited. Re-run: `adlc start PER-7 --skip-dev` or `adlc resume PER-7`.

## Mandatory scope

- Involved repo is only `agent`. Engine story id is only `01-agent`.
- Exactly **one** story LLD: `stories/01-agent.md`. Do not create or require `02-*` / `03-*` story files.
- Keep ticket scope unchanged:
  - `POST /query` with `{"query": "..."}` → `{"answer": "..."}`
  - LLM decides direct answer vs MCP tool call(s)
  - MCP tools over **stdio** only (PER-6 `python -m mcp.server`)
  - No FastMCP / no HTTP MCP server / no PER-6 transport changes
- Example must work in specs: `what is the name of employee with emp-001` → tool path with ID normalization to `EMP-001`.

## Exact test ownership (use everywhere — no placeholders)

Total **21** unit tests + **1** integration test + documented manual verification script.

| Area | File | Count | Must include |
|------|------|-------|--------------|
| HTTP server | `tests/test_http_server.py` | **6** | `/query` happy/validation/errors + **2** `/health` checks |
| LLM client | `tests/test_llm_client.py` | **3** | direct success, API error, env validation |
| MCP stdio client | `tests/test_mcp_client.py` | **4** | list/call tools, startup error, **timeout** |
| Orchestrator | `tests/test_orchestrator.py` | **8** | direct answer, tool success (employee lookup), MCP error, LLM call count (=2 on tool path), **ID normalization**, **arg validation missing field**, **arg validation unknown tool**, **synthesis loop detection** |

- `6 + 3 + 4 + 8 = 21` must appear consistently in requirements AC6, breakdown, HLD coverage, and `01-agent` Tests.
- Breakdown may describe narrative slices 1–5, but:
  - **Do not** say ID normalization / arg validation / synthesis loop / timeout are “placeholders” or “deferred to Story 5”.
  - Slice covering orchestrator owns all **8** orchestrator tests as **real** tests.
  - Slice covering MCP client owns all **4** MCP tests (including timeout) as **real** tests.
  - Final HTTP-only slice adds only the **2** health tests (HTTP total 6); it completes **no** orchestrator/MCP placeholders.

## Repair targets

1. Rewrite `breakdown.md` Story 1.4 and 1.5 (or equivalent) Verification Approach sections so they match the table above and the final note — remove contradictory deferrals.
2. Align `requirements.md` AC6 wording with the same ownership table.
3. Ensure `stories/01-agent.md` Tests + Implementation Details list the same 21 tests (same names/counts); no cutoffs.
4. Keep HLD outcomes binding (`/query`, LLM tool-calling, stdio MCP, ID `.upper()` on `id` args).

## Repair rules

- Keep ticket/repo labels (`repo:agent`) and out-of-scope constraints.
- Prefer one complete story LLD per involved repo; rewrite incomplete artifacts fully.
- Align AC counts, test counts, and file paths across requirements ↔ HLD ↔ breakdown ↔ stories.
- Critic must PASS when the 21-test ownership is consistent and placeholder/deferral contradictions are gone; do not FAIL solely because breakdown uses narrative slice numbering while only `01-agent.md` exists on disk.
