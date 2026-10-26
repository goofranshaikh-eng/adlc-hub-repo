# ADLC Consistency Review

FAIL

## Concrete Mismatches

1. **AC Count Misalignment (ticket vs. requirements.md vs. proposed_tickets.md)**
   - Ticket: 6 ACs (AC-1 through AC-6)
   - requirements.md: 6 ACs (AC-1 through AC-6) ✓ matches
   - proposed_tickets.md: 6 ACs split across 3 stories (AC-1,AC-2 → 01-agent; AC-3,AC-4,AC-5 → 02-agent; AC-6 → 03-agent) ✓ consistent
   - **Issue**: AC count is consistent, but ticket document structure differs slightly from requirements

2. **Test Count Mismatch (requirements.md Verify vs. breakdown.md vs. hld.md)**
   - requirements.md Verify section: States **32 tests total (8 + 15 + 9)**
   - breakdown.md Coverage table: Shows **8 + 15 + 9 = 32 tests** ✓ matches
   - hld.md Testing Summary: States **32 tests** ✓ matches
   - proposed_tickets.md: States **32 tests (8 + 15 + 9)** ✓ matches
   - **Issue**: All counts align at 32 tests; no mismatch

3. **Story LLD Test Counts vs. breakdown.md**
   - stories/01-agent.md: 8 tests (Tests 1–8) ✓ matches breakdown AC-1 (2 tests) + AC-2 (6 tests) = 8 total
   - stories/02-agent.md: 15 tests (Tests 1–15) ✓ matches breakdown AC-3 (7) + AC-4 (4) + AC-5 (3) = 14... **MISMATCH FOUND**
     - Breakdown shows: AC-3 (7 tests), AC-4 (4 tests), AC-5 (3 tests) = 14 tests total
     - stories/02-agent.md lists 15 tests in verification section (Tests 1–15)
     - **Resolution**: Test 6–8 (hold timing), Test 9–11 (failure), Tests 12–15 (stream closure, subscriber disconnect, late subscriber) = 10 tests covering AC-4 and AC-5; Test 1–5, 14–15 cover AC-3 (7 tests: HTTP 200, in_progress, same run_id, 404, multi-subscriber, disconnect, late replay). Recounting: AC-3 = 7, AC-4 = 4 (Tests 6–8, 12), AC-5 = 4 (Tests 9–11, 13). Total = 15. breakdown.md AC-5 row shows "3 tests" but should be "4 tests"
   - **Issue**: breakdown.md AC-5 row states "3 tests" but stories/02-agent.md and requirements.md indicate 4 tests (LLM error, MCP error, invalid query rejection, stream closure after failed)
   - stories/03-agent.md: 9 tests (Tests 1–9) ✓ matches breakdown AC-6 (9 tests)

4. **AC-5 Test Coverage Gap in breakdown.md**
   - breakdown.md table row for AC-5 lists "3 tests (Tests 9–11, 13)" but only counts 3
   - Should be "4 tests (Tests 9–11, 13)" to include stream closure after failed
   - stories/02-agent.md and hld.md both imply 4 tests for AC-5 (immediate failure handling + stream closure verification)
   - **Change_requests.md says**: "No new external API fields, types, or operations beyond the ticket, change_requests.md, and repo-context"
   - **Constraint violation**: This affects test count alignment between breakdown and story LLDs

5. **File Paths Consistency Check**
   - change_requests.md specifies: "Routes go in agent/main.py. Keep agent/orchestrator.py, agent/health.py, and agent/mcp_client.py. New modules: agent/models.py, agent/run_store.py, agent/sse_handler.py, agent/background_task.py."
   - proposed_tickets.md file list: ✓ all paths match
   - hld.md file list: ✓ all paths match
   - specs/PER-43.md file list: ✓ all paths match
   - specs/PER-44.md file list: ✓ all paths match
   - specs/PER-45.md file list: ✓ all paths match
   - breakdown.md implementation locations: ✓ all paths match
   - stories/01-agent.md, 02-agent.md, 03-agent.md: ✓ all paths match
   - **No file path mismatches found**

6. **Missing specs/<id>.md Files**
   - proposed_tickets.md references story IDs: 01-agent (PER-43), 02-agent (PER-44), 03-agent (PER-45)
   - specs/PER-43.md exists ✓
   - specs/PER-44.md exists ✓
   - specs/PER-45.md exists ✓
   - All three child tickets have specs files; no missing files

7. **HLD Outcome Format vs. LLD Outcomes**
   - hld.md describes outcomes in SSE event format and HTTP responses (generic, not language-specific) ✓
   - No language-specific syntax (Python f-strings, JSON encoder calls) in HLD ✓
   - LLDs specify Python Pydantic models and asyncio patterns (appropriate for story-level detail) ✓
   - No mismatch between HLD outcomes and LLD implementations

8. **Merged Plan Order (not present)**
   - No `merged_plan.md` file referenced or present
   - Not applicable for this project scope
   - No constraint violation

9. **Stack/Language Context**
   - change_requests.md specifies: "Use existing PER-20 layout. Routes go in agent/main.py."
   - All specs and stories use Python 3.11+ asyncio patterns ✓
   - No TypeScript, Node.js, or other language paths introduced ✓
   - Consistent with repo-context (Python agent project)

10. **AC Wording Alignment (ticket vs. requirements.md)**
    - Ticket AC-1: "Immediate accept" ✓ matches requirements.md AC-1 wording
    - Ticket AC-2: "Reject without a run" ✓ matches requirements.md AC-2 wording
    - Ticket AC-3: "One event stream" ✓ matches requirements.md AC-3 wording
    - Ticket AC-4: "Five-second hold on completed only" ✓ matches requirements.md AC-4 wording
    - Ticket AC-5: "Failure is immediate" ✓ matches requirements.md AC-5 wording
    - Ticket AC-6: "Existing `/query` path unchanged" ✓ matches requirements.md AC-6 wording
    - All AC phrasings consistent across ticket and requirements

## Primary Issue: Test Count Mismatch in breakdown.md

**Location**: breakdown.md, Story 2 Coverage & Verification table, AC-5 row

**Current state:**
```
| AC-5: Failure is immediate | Story 2 | 3 | Tests 9–11, 13: LLM/MCP errors, invalid query rejection, stream closes |
```

**Should be:**
```
| AC-5: Failure is immediate | Story 2 | 4 | Tests 9–11, 13: LLM/MCP errors, invalid query rejection, no hold, stream closes |
```

**Rationale**: 
- requirements.md Verify section lists: "Failure is immediate: 3 tests" — but the description "LLM/MCP errors sent immediately as failed, invalid query rejection as failed event, stream closure after completed or failed" implies 4 distinct test scenarios
- stories/02-agent.md Verification Approach table lists 15 tests (not 14), including:
  - Test 9: LLM error immediate
  - Test 10: MCP error immediate
  - Test 11: Invalid query rejection
  - Test 13: Stream closes after failed
  - Plus Test 12 (stream closes after completed, part of AC-4)
- The correct count for AC-5 is **4 tests** (Tests 9–11 cover error types; Test 13 verifies no extra hold and stream closure)

**Resolution required:**
- Update breakdown.md AC-5 row test count from "3" to "4"
- Verify consistency across all remaining references (all other documents already show 15 tests for Story 2, indicating implicit 4-test count for AC-5)

---

## Summary

**FAIL reason**: breakdown.md Coverage & Verification table shows AC-5 test count as "3 tests" when it should be "4 tests" to align with:
- requirements.md: 15 tests total for Story 2 (Story 2 = AC-3 + AC-4 + AC-5 = 7 + 4 + 4 = 15)
- stories/02-agent.md: 15 total tests with 4 tests covering AC-5
- hld.md: Implies 15 tests for Story 2

All other consistency checks pass: AC counts, file paths, stack context, HLD outcomes, and specs file presence are aligned across ticket, requirements, HLD, breakdown, and story LLDs.
