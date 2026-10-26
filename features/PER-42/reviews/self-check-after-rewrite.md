FAIL

## Concrete Mismatches

### File Path Inconsistencies
- **breakdown.md and stories specify** `agent/src/agent/routes.py` for route handlers
- **specs/1.md, specs/2.md specify** `agent/src/routes/agent.py` (modularized under `routes/` subdirectory)
- **Mismatch**: Path conflicts between breakdown/stories (`agent/src/agent/routes.py`) and specs (`agent/src/routes/agent.py`). Consolidate to one canonical path across all artifacts.

### Missing Specification Files for Child Tickets
- **tickets.md maps**:
  - `01-agent` → `PER-43`
  - `02-agent` → `PER-44`
  - `03-agent` → `PER-45`
- **Artifacts provided**: `specs/1.md`, `specs/2.md`, `specs/3.md` (for tickets 1, 2, 3)
- **Issue**: No `specs/PER-43.md`, `specs/PER-44.md`, `specs/PER-45.md` file path alignment. Child ticket IDs should either use numeric references (1, 2, 3) or explicit ticket IDs (PER-43, PER-44, PER-45) consistently. Currently mixed notation makes traceability ambiguous.

### Test Count Discrepancies
- **breakdown.md specifies**:
  - Story 1: 8 tests (AC-1: 2, AC-2: 6)
  - Story 2: 15 tests (AC-3: 7, AC-4: 4, AC-5: 3 + closure tests)
  - Story 3: 9 tests (AC-6: 9)
  - **Total: 32 tests**
- **stories/01-agent.md, stories/02-agent.md, stories/03-agent.md** each list the same test counts (8, 15, 9)
- **Verification**: Test counts align across all artifacts ✓ (all show 32 total)
- **Note**: No mismatch here; counts are consistent.

### AC Coverage Mismatch in Breakdown vs. HLD
- **HLD "Story Breakdown" section** references `stories/01-agent.md` and `specs/1.md` with "see `breakdown.md` for…" forward reference
- **But HLD does NOT specify concrete story file paths in its "Acceptance Criteria Coverage" table** — only says "Story 1, Story 2, Story 3"
- **Breakdown.md DOES provide** concrete file paths: `stories/01-agent.md`, `specs/1.md`, etc.
- **Result**: HLD lacks concrete file paths; users must refer to breakdown.md to find story locations. This is not a mismatch (HLD → breakdown is allowed per ADLC rules), but inconsistent specificity.

### Route Handler File Path Conflict (CRITICAL)
- **specs/1.md, specs/2.md state**: `agent/src/routes/agent.py` (line: "2. **`agent/src/routes/agent.py`** — SSE stream reader & event formatter; yields `text/event-stream` format.")
- **breakdown.md, stories/01-agent.md, stories/02-agent.md state**: `agent/src/agent/routes.py` (line: "4. **`agent/src/agent/routes.py`** — FastAPI route handler for `POST /agent/query`.")
- **HLD does not specify file path** (intentional per ADLC rule: "Describe outcomes in HLD, not file paths")
- **Mismatch**: Two different paths used across specs and breakdown/stories. **Must consolidate to ONE canonical path.** Recommend `agent/src/agent/routes.py` (co-locating routes with models, run_store, etc. in the `agent/` package).

### Missing File in Breakdown Touch List for Story 2
- **breakdown.md "File Touch List" for Story 2** lists:
  ```
  agent/src/agent/run_store.py (extend with cleanup method…)
  agent/src/agent/sse_handler.py (SSE stream generator…)
  agent/src/agent/background_task.py (background orchestrator…)
  agent/src/agent/routes.py (extend with GET /agent/runs/…)
  agent/src/main.py (register cleanup task…)
  ```
- **specs/2.md "File Touch List"** lists the same paths **but adds**:
  ```
  agent/src/routes/__init__.py (NEW)
  agent/src/routes/agent.py (POST /agent/query, GET /agent/runs/{id}/events)
  ```
- **Mismatch**: specs/2.md introduces a modularized `routes/` subdirectory absent from breakdown.md. **Consolidate: either keep routes in `agent/src/agent/routes.py` (per breakdown/stories) or create `agent/src/routes/agent.py` (per specs), but not both.**

### Acceptance Criteria Wording Consistency
- **ticket AC-2**: "The LLM and MCP are not called."
- **requirements.md AC-2**: "the LLM and MCP are not called"
- **All stories/specs**: Match ticket wording ✓
- **No mismatch here.**

### Schema & API Contracts Alignment
- **HLD Contract section** specifies request/response shapes (JSON).
- **specs/1.md, specs/2.md** provide Pydantic model definitions.
- **breakdown.md** references "see specs/ for schema."
- **All align** ✓

### Event Timing in AC-4 (Fast Answer Hold Logic)
- **ticket AC-4**: "If the answer is ready less than 5 seconds after `in_progress`, the stream stays on `in_progress` until 5 seconds after that event, then sends `completed`…"
- **HLD Algorithm section** (5-Second Hold Enforcement): Shows timeline with "t=5.5" completion.
- **specs/2.md Background Task Algorithm**: Shows same calculation: `hold_remaining = 5.0 - elapsed`.
- **All align** ✓

### Definition of Done Coverage
- **ticket DoD**: "Tests cover immediate return, rejection, the fast and slow completed cases, a tool or LLM failure, an unknown run id, and the existing `POST /query` answer."
- **breakdown.md Acceptance Criteria Coverage table**: All 6 ACs covered; test counts align.
- **stories (01, 02, 03)**: Tests listed match breakdown counts.
- **All align** ✓

---

## Required Fixes

1. **Resolve route file path conflict**: Pick ONE canonical path—recommend `agent/src/agent/routes.py` (used by breakdown and stories). Update **specs/1.md and specs/2.md** to use `agent/src/agent/routes.py` instead of `agent/src/routes/agent.py`. Remove the separate `agent/src/routes/` directory from all file touch lists.

2. **Align child ticket ID notation**: Use consistent notation throughout. Either:
   - Use numeric IDs (1, 2, 3) and reference `specs/1.md`, `stories/01-agent.md`, or
   - Use full ticket IDs (PER-43, PER-44, PER-45) and reference `specs/PER-43.md`, `stories/PER-43-agent.md`.
   - Currently mixed; choose one and update **tickets.md, breakdown.md, HLD**, and all story/spec files.

3. **HLD Story Breakdown section**: Add concrete file path pointers (e.g., "See `stories/01-agent.md` and `specs/1.md`…") to match the specificity in breakdown.md.

4. **Remove modularized routes directory from specs**: Update **specs/1.md and specs/2.md "File Touch List"** section to remove:
   ```
   agent/src/routes/
   ├── __init__.py
   └── agent.py
   ```
   and consolidate all route handlers into `agent/src/agent/routes.py`.

---

**Root Cause**: Specs were drafted with an alternate directory structure (`src/routes/agent.py`) while breakdown and stories assume co-location (`src/agent/routes.py`). Resolve before implementation begins to avoid mid-build refactoring.
