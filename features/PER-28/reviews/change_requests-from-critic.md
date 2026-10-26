# Course correction PER-28 (auto from critic exhaustion)

Authoritative for the next run. Human: review/edit this file, then re-run:
`adlc start PER-28 --skip-dev` or `adlc review request-changes PER-28 --note "..."` then `adlc resume PER-28`.

## Critic mismatches to resolve

- **AC count mismatch (requirements vs breakdown)**: Requirements has 5 ACs (AC1–AC5). Breakdown claims to cover all 5, but Story 1 only covers AC1 and AC2 (schema definition), Story 2 covers AC1, AC4, and AC5, Story 3 covers AC3, Story 4 covers AC1, and Story 5 covers AC2, AC3, and AC4. The per-story AC coverage is inconsistent with the breakdown's summary table, which lists which stories cover which ACs but does not match the story LLDs' actual AC sections.

- **Test count mismatch (breakdown vs stories)**: Breakdown claims 27 total tests (Story 1: 4, Story 2: 6, Story 3: 5, Story 4: 4, Story 5: 8). Story LLDs report: Story 1: 4 tests, Story 2: 6 tests, Story 3: 5 tests, Story 4: 4 tests, Story 5: 8 tests. Total: 27. This matches, so no mismatch here.

- **Missing specs/<id>.md**: The breakdown lists six stories (01-fastapi through 06-fastapi), but only five specs exist (specs/1.md, stories/01-fastapi.md through stories/05-fastapi.md, and stories/06-fastapi.md which is marked as merged into Story 5). The proposed_tickets.md lists only one child ticket (ticket 1 covering AC1–AC5). The tickets.md file maps 01-fastapi → PER-29, 02-fastapi → PER-30, 03-fastapi → PER-31, 04-fastapi → PER-32, 05-fastapi → PER-33, but there is no 06-fastapi → PER-34 mapping shown in tickets.md, even though stories/06-fastapi.md exists and claims ticket PER-34. This is a missing mapping in tickets.md.

- **File paths mismatch (breakdown vs stories)**: Breakdown lists "app/services/run_state_manager.py" in Story 3 and references a `RunStateManager` component in the HLD, but change_requests.md explicitly removes this component (no timer logic, no status tracking). The HLD was updated to remove `RunStateManager` and replace it with `RunRegistry`, and the breakdown notes section says "Removed `RunStateManager` (Stories 3 and 5-second delay logic) per `change_requests.md`". However, specs/1.md (the top-level spec) still lists `app/services/run_state_manager.py` in the file touch list and describes `RunStateManager` in the component breakdown section. This contradicts change_requests.md and the updated breakdown/stories. The spec must be updated to remove `RunStateManager` and replace it with `RunRegistry`.

- **HLD outcome vs LLD mismatch**: The HLD's "Component Interactions" sequence diagram shows FastAPI creating and delaying status events (timer logic, 5-second delay before sending `completed`). The change_requests.md explicitly removes this: "FastAPI must not create, decide, or delay run status. The agent owns all timing logic, including the 5-second delay." The breakdown and stories correctly reflect this change (no timer logic, pure pass-through), but the HLD's sequence diagram and component descriptions still describe the old behavior. The HLD must be updated to remove the timer logic and 5-second delay references.

- **Contradicts change_requests.md**: specs/1.md describes the 5-second delay logic in detail ("Critical Logic: 5-Second Delay" section) and includes a pseudocode implementation that waits until T+5s before sending the `completed` event. This contradicts change_requests.md, which states "FastAPI must not create, decide, or delay run status. The 5 second timing stays on the agent, in a later ticket." The spec must remove this section and replace it with a note that the agent owns all timing logic.

## Repair rules

- Keep ticket/repo labels (`repo:*`) and out-of-scope constraints.
- Prefer one complete story LLD per involved repo when possible; do not leave truncated mid-sentence files.
- Align AC counts, test counts, and file paths across requirements ↔ HLD ↔ breakdown ↔ stories.
- Rewrite incomplete artifacts fully (no cutoffs).
