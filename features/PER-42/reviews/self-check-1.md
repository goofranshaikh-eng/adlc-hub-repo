FAIL

## Concrete Mismatches

1. **File paths mismatch**: `specs/1.md`, `specs/2.md`, `specs/3.md` reference `agent/src/agent/routes.py` and `agent/src/routes/agent.py`, but `change_requests.md` explicitly states "Do not add agent/src/agent/routes.py or agent/src/routes/agent.py" and "Routes go in agent/main.py."

2. **Spec organization mismatch**: The breakdown and stories reference specs at `specs/1.md`, `specs/2.md`, `specs/3.md`, but the ticket references `specs/PER-43.md`, `specs/PER-44.md`, `specs/PER-45.md` (which exist in the provided context). The traceability map is ambiguous: should child tickets reference `specs/1.md` or `specs/PER-43.md`?

3. **Background task interface incomplete**: Stories 1–3 and specs/1.md define `spawn_background_job()` as placeholder in `background_task.py`, but `change_requests.md` and the ticket require it to call `process_question` and enqueue events. The full implementation is deferred to Story 2, but Story 1's interface contract does not specify the event enqueue signature or error handling. This creates a gap: Story 1 tests (test_agent_query_accept.py) cannot verify that the spawned job will eventually emit events without duplicating Story 2 logic.

4. **Test count mismatch in stories vs. breakdown**: 
   - `breakdown.md` lists Story 1: "8 verification tests", Story 2: "15 verification tests", Story 3: "9 verification tests" (total 32).
   - `stories/01-agent.md` Test section shows 8 tests (✓).
   - `stories/02-agent.md` Test section shows 15 tests (✓).
   - `stories/03-agent.md` Test section shows 9 tests (✓).
   - Consistent across all three files.

5. **Spec/story file path inconsistency**: `specs/1.md` (Story 1) references file touch list with `agent/src/agent/routes.py`, but `change_requests.md` prohibits it. `stories/01-agent.md` correctly lists only `agent/src/agent/models.py`, `agent/src/agent/run_store.py`, `agent/src/agent/background_task.py`, and `agent/src/main.py` (no routes.py). Stories are correct; specs are wrong.

6. **Model definitions mismatch**: `specs/2.md` and `specs/3.md` include Pydantic model definitions (e.g., `InProgressEventData`), but `specs/1.md` does not, even though it defines the request/response models for POST /agent/query. Story 1 should define all models used in Story 1; models defined in Story 2 should be in specs/2.md only (no duplication).

7. **PER-43, PER-44, PER-45 vs. 1, 2, 3 naming**: Tickets.md maps child tickets as `01-agent → PER-43`, `02-agent → PER-44`, `03-agent → PER-45`. But `specs/PER-43.md`, `specs/PER-44.md`, `specs/PER-45.md` exist and contain the full implementations. The breakdown references `specs/1.md`, `specs/2.md`, `specs/3.md`, not the PER- numbered files. This creates two parallel spec hierarchies. Clarification needed: should breakdown point to `specs/PER-43.md` or keep `specs/1.md` as the source of truth for Story 1?

8. **Change request not applied to specs/1.md**: `change_requests.md` says "Routes go in agent/main.py" and "Do not add agent/src/agent/routes.py or agent/src/routes/agent.py," but `specs/1.md` still lists both in the "Allowed Paths" section:
   ```
   - `agent/src/agent/models.py` — Pydantic models
   - `agent/src/agent/run_store.py` — run registry
   - `agent/src/agent/orchestrator.py` — background job spawn logic
   - `agent/src/main.py` — register agent router (POST /agent/query)
   ```
   Correct (no routes.py). But the text says "register agent router," which is ambiguous: is the router defined in `main.py` or imported? `specs/PER-43.md` and `stories/01-agent.md` clarify it's in `main.py`. `specs/1.md` does not.

9. **RunStore.cleanup_expired() implementation deferred but not marked**: `specs/1.md` defines `cleanup_expired()` as `pass` ("deferred to Story 2"), but the signature is present. `specs/2.md` provides the implementation. This is acceptable (forward reference), but `specs/1.md` should note "See Story 2 for implementation" explicitly (currently just says `pass` without comment).

10. **AC-2 event enqueue timing unclear in Story 1 spec**: `specs/1.md` rejection path says "call `run_store.mark_failed(agent_run_id, "invalid query")` to enqueue `failed` event before returning," but the interface signature for `mark_failed()` shows it enqueues both `in_progress` (already enqueued by `create()`) and `failed`. This is correct per the Run dataclass, but the narrative in `specs/1.md` is ambiguous: does `create()` enqueue in_progress or does `mark_failed()` do it? `specs/PER-43.md` clarifies: `create()` enqueues in_progress immediately; `mark_failed()` enqueues failed. `specs/1.md` should match.

11. **Background task spawn interface mismatch**: `specs/1.md` defines `spawn_background_job()` signature as `async def spawn_background_job(agent_run_id: str, query: str) -> None`, but the route handler passes `run_store` as a third argument in `stories/01-agent.md` code example: `asyncio.create_task(spawn_background_job(run.id, query, run_store))`. The interface signature is incomplete; should be `async def spawn_background_job(agent_run_id: str, query: str, run_store: RunStore) -> None`.

12. **Cleanup task initialization timing**: `specs/2.md` and `stories/02-agent.md` define `cleanup_task()` to run on app startup via `@app.on_event("startup")`. The implementation is correct. However, `specs/1.md` does not mention startup events or the cleanup task, which is deferred to Story 2. This is acceptable (Story 1 doesn't require cleanup logic), but the narrative in specs/2.md should note that startup is added in Story 2, not Story 1.

---

**Summary**: The artifacts contain multiple path and interface inconsistencies introduced by the change request (prohibiting separate routes files) not being fully applied to specs/1.md, specs/2.md, specs/3.md. The PER-43/44/45 vs. 1/2/3 dual numbering creates ambiguity about which spec files are the source of truth. Story-level files (`stories/01-agent.md`, etc.) are mostly consistent with the change request and each other; spec files contain outdated or inconsistent information.
