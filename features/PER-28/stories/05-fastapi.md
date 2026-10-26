# Story 05-fastapi LLD: SSE Stream & Cleanup

**Ticket:** PER-33  
**Story:** 05-fastapi  
**Repo:** See LLD (stack/paths deferred to implementation based on repo context)  
**Status:** Draft

---

## Overview

Implement GET /query/{run_id}/stream endpoint as a pure SSE event forwarder. The endpoint subscribes to the agent's event stream via `AgentClient` and forwards events (`in_progress`, `completed`, `failed`) to the client in real-time. Closes the stream after a terminal event and cleans up the run registry mapping. Includes run_id validation, agent subscription timeout handling, and client disconnection detection.

**CRITICAL CHANGE (per change_requests.md):** FastAPI does **not** create, decide, or delay run status. The agent owns all timing logic, including the 5-second delay. This endpoint is a pure pass-through: it forwards the agent's events exactly as received, with no timer logic, no `in_progress`/`completed`/`failed` generation except in error/timeout scenarios (see AC3 Clarification below).

---

## Acceptance Criteria

### AC2: SSE stream forwarding
**Given** a query has been submitted  
**When** the SSE stream is opened for that run  
**Then** the stream forwards the status and response events the agent already sent

### AC3: No status creation
**Given** the integration is running  
**When** processing requests  
**Then** FastAPI does not create, decide, or delay run status (no timer logic, no in_progress/completed/failed generation)

**Clarification (per comment 4374346576):** AC3 prohibits artificial timing logic and normal-flow status generation. Exception: When agent connection times out or fails mid-stream (agent already sent in_progress), backend generates `failed` event as documented in HLD Risk 3 mitigation. This is backend recovery, not normal forwarding. Tests must distinguish agent-forwarded events from backend-generated exception events.

### AC4: No polling behavior
**Given** the integration is running  
**When** waiting for agent responses  
**Then** FastAPI does not poll the agent for status or responses

**Clarification (per comment 4374346572):** Agent disconnection during non-terminal event (e.g., mid-in_progress) is treated as stream error, not retried. Single subscription per run guaranteed; no polling backoff or retry loops.

---

## Scope

This story defines the SSE stream endpoint interface and behavior contracts. Concrete file paths, language-specific implementations, and stack choices are deferred to implementation based on actual repo context, which currently shows "(repo path not mapped; do not invent a stack)" per `repo-tree/fastapi`.

---

## Functional Requirements

### GET /query/{run_id}/stream Endpoint

**Responsibility:** Forward agent's SSE event stream to client.

**Interface:**
```
GET /query/{run_id}/stream
Accept: text/event-stream
Response: text/event-stream with events:
  - in_progress: {"run_id": "<uuid>"}
  - completed: {"run_id": "<uuid>", "response": {...}}
  - failed: {"run_id": "<uuid>", "error": "<message>"}
HTTP 200 (stream established) or 404 (run_id not found)
```

**Behavior:**
- Validates run_id exists in registry
- Looks up agent_run_id from registry
- Subscribes to agent's event stream (single connection, no polling)
- Forwards events as received (pure pass-through)
- Closes stream after exactly one terminal event (completed or failed, never both)
- On agent timeout or error: generates `failed` event (backend exception recovery, not normal forwarding)
- Cleans up registry mapping in finally block
- NO artificial timer logic, NO scheduled delays, NO normal-flow status generation

**Response Codes:**
- `200 OK` with `Content-Type: text/event-stream`: SSE stream established
- `404 Not Found`: run_id not found in registry (query not submitted or already cleaned up)

**AC Coverage:** AC2 (forwards agent events), AC3 (no status creation except error recovery), AC4 (single subscription)

---

## Endpoint Algorithm

**Processing Flow:**
1. Validate run_id exists in registry
2. Lookup agent_run_id from registry
3. Subscribe to agent's event stream (single connection)
4. For each agent event received:
   - Forward event exactly as received to SSE stream
   - If event is terminal (completed or failed), break (do not process further events)
5. Close SSE stream
6. Clean up registry mapping in finally block

**Key Constraints:**
- Pure pass-through: forwards agent events exactly as received
- No timer logic or scheduled delays: events streamed when received
- Single terminal event guarantee: stream closes after first completed/failed, no further events forwarded
- Single subscription: one connection to agent per SSE stream (no polling, no retries)
- Cleanup always runs: registry mapping removed in finally block, even on client disconnect or timeout

**Error Handling:**
- Run_id not found in registry: HTTP 404 "Run not found"
- Agent subscription timeout (5 minutes): Generate and send SSE failed event with "Agent timeout" (backend exception event)
- Agent subscription error (e.g., connection drop during non-terminal): Generate and send SSE failed event with "Stream error" (backend exception event)
- Client disconnection: Cleanup runs, agent subscription cancelled, no event generated

---

## SSE Event Format

**Standard SSE Format:**
```
event: <type>
data: <json>

```

**Event Types:**
- `in_progress`: Run started, no response yet (agent-forwarded)
- `completed`: Run finished successfully, includes response (agent-forwarded)
- `failed`: Run failed, includes error message (agent-forwarded or backend-generated on timeout/error)

**Example Events (Agent-Forwarded):**
```
event: in_progress
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000"}

event: completed
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000", "response": {"result": "success"}}

event: failed
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000", "error": "agent internal error"}
```

**Example Event (Backend-Generated Exception):**
```
event: failed
data: {"run_id": "123e4567-e89b-12d3-a456-426614174000", "error": "Agent timeout"}
```

---

## Dependencies

**Component Dependencies:**
- Story 2: AgentClient (subscribe_to_run method)
- Story 3: RunRegistry (get_agent_run_id, remove_run methods)
- Story 4: POST /query endpoint (creates run registrations)

**Runtime Requirements:**
- SSE streaming library (framework-dependent)
- Async runtime support
- JSON serialization (typically stdlib)

**Test Requirements:**
- Test framework with async support
- SSE client test utilities
- Mocking library

---

## Implementation Guidance

When implementing this story in the actual codebase:

1. **Choose SSE library** based on repo stack (sse-starlette for FastAPI, EventSource wrapper for Node.js, etc.)
2. **Locate router/controller module** from Story 4
3. **Define endpoint handler** matching the interface contracts above
4. **Implement event generator** as async iterator/generator
5. **Use dependency injection** for AgentClient and RunRegistry (from Story 4)
6. **Handle errors** per the error translation strategy below
7. **Enforce terminal event guarantee:** Break immediately after forwarding first completed/failed event; do not iterate further on agent stream
8. **Document exception events:** Code comments distinguish backend-generated `failed` events (timeout, connection error) from agent-forwarded events

**Key Principles:**
- The contracts above (operation signatures, event formats, error cases) are fixed
- The implementation syntax (language, SSE library, async patterns) adapts to the repo's stack
- No timer logic, no scheduled delays — only event forwarding and exception recovery
- Terminal event guarantee: stream closes after exactly one of completed/failed, never both

---

## Algorithm Pseudocode

```
function stream_status(run_id, agent_client, registry):
    # Validate run_id exists
    try:
        agent_run_id = registry.get_agent_run_id(run_id)
    except KeyError:
        raise HTTPException(404, "Run not found")
    
    # Define SSE event generator
    async function event_generator():
        try:
            # Subscribe to agent event stream (single connection, no polling)
            async for agent_event in agent_client.subscribe_to_run(agent_run_id):
                # Forward event exactly as received (pure pass-through)
                event_type = agent_event["event"]
                event_data = agent_event["data"]
                
                # Serialize to SSE format
                sse_message = format("event: {event_type}\ndata: {json(event_data)}\n\n")
                yield sse_message
                
                # Enforce terminal event guarantee: close after first completed/failed
                if event_type in ("completed", "failed"):
                    # Do not process further events from agent stream
                    break
        
        except TimeoutError:
            # Backend exception event: agent subscription timeout (5 minutes)
            # Not a forwarded agent event; generated here as error recovery
            yield format("event: failed\ndata: {{\"run_id\": \"{run_id}\", \"error\": \"Agent timeout\"}}\n\n")
        
        except Exception:
            # Backend exception event: agent subscription error (connection drop, etc.)
            # Not a forwarded agent event; generated here as error recovery
            yield format("event: failed\ndata: {{\"run_id\": \"{run_id}\", \"error\": \"Stream error\"}}\n\n")
        
        finally:
            # Cleanup registry mapping (prevent memory leak)
            # Runs even on client disconnect
            try:
                registry.remove_run(run_id)
            except KeyError:
                pass  # Already cleaned up
    
    # Return SSE stream response
    return SSEResponse(event_generator())
```

**Key Points:**
- Single subscription per SSE stream (no polling, no retries)
- Agent-forwarded events passed through exactly as received
- Terminal event guarantee: break after first completed/failed, do not yield further events
- Backend-generated exception events (timeout, stream error) distinguished by source and context
- Cleanup in finally block (always runs, even on client disconnect)
- Client disconnection detected automatically by SSE library; triggers cleanup via task cancellation

---

## Error Translation Strategy

| Error | SSE Event | HTTP Status | Event Source | Notes |
|-------|-----------|-------------|--------------|-------|
| `KeyError` from `registry.get_agent_run_id()` | None | 404 | N/A (endpoint error) | Run not found (query not submitted or already cleaned up); no stream established |
| Timeout from `agent_client.subscribe_to_run()` | `failed` with "Agent timeout" | 200 (stream established) | Backend exception recovery | Agent took > 5 minutes; stream already sent in_progress; backend generates failed as recovery |
| HTTP/connection error from `agent_client.subscribe_to_run()` | `failed` with "Stream error" | 200 (stream established) | Backend exception recovery | Agent subscription failed mid-stream (connection drop, etc.); backend generates failed as recovery |
| Client disconnects mid-stream | None (connection closed) | N/A (connection closed) | Client action | SSE library detects disconnect; cleanup still runs in finally block |

**Rationale:**
- 404 for missing run_id (client error: wrong ID or expired run); no stream established
- Once stream established (200), errors sent as SSE failed events
- Backend-generated exception events (timeout, stream error) only occur after agent has sent at least one event (in_progress) and then failed; these are recovery, not normal forwarding
- Client disconnection: SSE library handles; no explicit error event needed; cleanup guaranteed by finally block

---

## Tests

### Test 1: SSE Stream Forwards Agent Events (Integration)

**Verification:** requirements.md AC2 (SSE stream forwards status and response events the agent already sent)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → completed
- Connect to SSE endpoint
- Assert 200 status, Content-Type: text/event-stream
- Assert 2 events received: in_progress, completed
- Assert event data matches agent events exactly (forwarded, not transformed)
- Assert registry cleanup called

---

### Test 2: SSE Stream Forwards Failure Event (Integration)

**Verification:** requirements.md AC2 (SSE stream forwards agent events), AC3 (FastAPI does not create status events in normal flow)

**Acceptance Criteria:** AC2 (SSE stream forwarding), AC3 (no status creation)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → failed (agent-generated failure)
- Connect to SSE endpoint
- Assert 2 events received: in_progress, failed
- Assert failed event is agent-sourced (verify via mock setup: agent yielded this event)
- Assert completed event never sent
- Assert registry cleanup called

---

### Test 3: SSE Stream Closes After Completed Event (Integration)

**Verification:** requirements.md AC2 (SSE stream closes after terminal event), HLD AD-4 (exactly one terminal event)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → completed → unexpected_event
- Connect to SSE endpoint
- Assert only 2 events received (unexpected_event not forwarded)
- Assert stream closed after completed
- Assert no further iteration on agent stream (agent subscription cancelled/closed)

---

### Test 4: SSE Stream Closes After Failed Event (Integration)

**Verification:** requirements.md AC2 (SSE stream closes after terminal event), HLD AD-4 (exactly one terminal event)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → failed → unexpected_event
- Connect to SSE endpoint
- Assert only 2 events received (unexpected_event not forwarded)
- Assert stream closed after failed
- Assert no further iteration on agent stream (agent subscription cancelled/closed)

---

### Test 5: SSE Stream Returns 404 for Missing Run ID (Unit)

**Verification:** requirements.md AC2 (validates run_id before forwarding)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock registry raises KeyError for run_id lookup
- Attempt to connect to SSE endpoint
- Assert 404 response with "Run not found" detail
- Assert agent client never called

---

### Test 6: SSE Stream Timeout Sends Backend-Generated Failed Event (Integration)

**Verification:** requirements.md AC2 (handles agent timeout), AC3 (backend-generated exception event is error recovery, not normal status creation)

**Acceptance Criteria:** AC2 (SSE stream forwarding), AC3 (clarified exception handling)

**Test Coverage:**
- Mock agent stream yielding: in_progress → timeout error
- Connect to SSE endpoint
- Assert 2 events received: in_progress, failed
- Assert failed event contains "Agent timeout" error
- **Assert failed event is backend-generated** (not from agent mock; generated by exception handler per pseudocode)
- Verify in code comment or test assertion: this failed event is exception recovery, distinct from AC3's prohibition on normal-flow status creation

---

### Test 7: SSE Client Disconnection Triggers Cleanup (Integration)

**Verification:** requirements.md AC2 (cleanup on disconnection), per comment 4374346590

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Mock long-running agent stream (yields in_progress, then waits indefinitely)
- Connect to SSE endpoint
- Receive first event (in_progress)
- Disconnect SSE client mid-stream
- Assert registry cleanup called despite client disconnection
- Assert agent subscription is cancelled (no further iteration attempted)

---

### Test 8: Multiple Runs Cleanup After Terminal Events (Integration)

**Verification:** requirements.md AC2 (no memory leak)

**Acceptance Criteria:** AC2 (SSE stream forwarding)

**Test Coverage:**
- Submit 10 queries (POST /query) via Story 4 endpoint
- Connect to 10 SSE streams (GET /query/{run_id}/stream)
- Mock agent to yield in_progress → completed for each
- Consume all streams fully
- Assert all 10 registry mappings cleaned up
- Verify registry size decreases to 0 after all streams close

---

### Test 9: Stream Stops After First Terminal Event (New, per comment 4374346586)

**Verification:** HLD AD-4 (exactly one terminal event; never send both completed and failed)

**Acceptance Criteria:** AC2 (SSE stream forwarding), AC3 (terminal event guarantee)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → completed → failed (malformed: sends both terminal events)
- Connect to SSE endpoint
- Assert exactly 2 events received: in_progress, completed
- Assert failed event (the second one) is NOT forwarded
- Assert stream closed after first terminal event
- Assert no further agent stream iteration (agent subscription cancelled immediately after completed)

---

### Test 10: Agent Disconnection During Non-Terminal (New, per comment 4374346572)

**Verification:** AC4 clarification (agent disconnect mid-stream is error, not retried)

**Acceptance Criteria:** AC4 (no polling behavior)

**Test Coverage:**
- Mock agent event stream yielding: in_progress → connection drop (no more events, stream ends)
- Connect to SSE endpoint
- Assert in_progress event received
- Assert backend generates failed event with "Stream error" (exception recovery)
- Verify no retry loop, no polling: subscription closes after first disconnect
- Assert no additional HTTP calls to agent

---

## Open Questions

1. **SSE reconnection:** Should clients be able to reconnect mid-stream with `Last-Event-ID` header? Current design: no reconnection support (each stream is one-shot).

2. **Authentication on SSE endpoint:** Should SSE stream require authentication? Current design: no auth (if POST /query was authenticated, assume run_id is scoped per user).

3. **SSE event IDs:** Should we add `id: <sequence>` field to SSE events for client reconnection? Current design: no IDs (simpler implementation).

4. **Agent event schema validation:** Should we validate agent events before forwarding? Current design: forward as-is (trust agent contract).

5. **Multiple clients per run_id:** Can multiple clients connect to same SSE stream? Current design: yes (agent_run_id allows multiple subscriptions). Acceptable behavior?

6. **Backend exception event granularity:** Should "Stream error" message include more detail (e.g., HTTP status code from agent)? Current design: generic message for simplicity; detailed errors logged server-side only.

7. **Stack confirmation:** Current repo-tree shows "(repo path not mapped; do not invent a stack)". Before implementing, confirm:
   - Language and web framework
   - Existing SSE library patterns
   - Async patterns and conventions
   - Router/controller organization

---

## Risks

### Risk 1: SSE client compatibility
**Impact:** Some clients (proxies, old browsers) may not support SSE or may buffer events  
**Likelihood:** Medium  
**Mitigation:** Document SSE requirements. Test with common clients. Consider WebSocket fallback if SSE proves unreliable.

### Risk 2: Agent event stream disconnection
**Impact:** SSE stream stalls, client never receives terminal event  
**Likelihood:** Medium  
**Mitigation:** 5-minute timeout on agent subscription. If agent disconnects, backend generates failed event and closes stream.

### Risk 3: Memory leak if cleanup fails
**Impact:** Registry grows unbounded  
**Likelihood:** Low (simpler than old design)  
**Mitigation:** Cleanup in finally block (always runs). No periodic cleanup needed (no timestamps to leak). Test 8 verifies registry size decreases to 0.

### Risk 4: SSE library buffering
**Impact:** Events not streamed in real-time (buffered until threshold reached)  
**Likelihood:** Low (SSE libraries designed for streaming)  
**Mitigation:** Test with real agent to verify latency. If buffering occurs, adjust library settings.

### Risk 5: Client disconnection not detected
**Impact:** Registry entry not cleaned up, memory leak  
**Likelihood:** Low (SSE libraries handle disconnection)  
**Mitigation:** Test disconnection scenario (Test 7). Verify cleanup via task cancellation in finally block. If disconnection not detected, consider explicit ping/pong mechanism.

### Risk 6: Malformed agent stream (multiple terminal events)
**Impact:** Both completed and failed sent to client, violating AC2/AC3  
**Likelihood:** Low (agent should be well-behaved)  
**Mitigation:** Terminal event guarantee enforced in pseudocode (break after first completed/failed). Test 9 verifies guarantee even for malformed streams.

---

## Notes

- **No timer logic (critical):** Per change_requests.md, FastAPI does **not** create, decide, or delay run status in normal flow. The agent owns all timing logic. This endpoint forwards agent events exactly as received. Exception: When agent connection fails (timeout, drop), backend generates `failed` event as error recovery, not normal forwarding.

- **AC3 Clarification:** AC3 prohibits "create, decide, or delay run status (no timer logic, no in_progress/completed/failed generation)". Normal flow: all events are agent-forwarded (no creation). Error recovery: on agent timeout or connection error, backend generates `failed` event. This is intentional per HLD Risk 3 mitigation. Tests must distinguish these two event sources (see Test 6 and Test 9 comments).

- **Terminal event guarantee:** HLD AD-4 mandates "After forwarding `completed` or `failed`, stream closes; never send both". Implementation enforces via break statement after first terminal event. Stream closed, no further iteration on agent stream. Test 9 verifies this guarantee even for malformed agent streams that violate the contract.

- **No polling (critical):** Per AC4, single subscription per run (no polling, no retries). Agent client creates one connection; endpoint breaks after first terminal event or error. Test 10 verifies disconnection mid-stream is error (generates failed), not retried.

- **Cleanup always runs:** Finally block ensures registry cleanup even if client disconnects (per Test 7) or agent times out (per Test 6). No stale mappings left behind.

- **Exception events vs. forwarded events:** Code must distinguish:
  - **Forwarded:** in_progress, completed, failed — originate from agent; passed through unchanged
  - **Backend-generated:** failed (on timeout/error) — generated by exception handlers; indicate backend recovery, not agent status
  - Tests verify event source via mock setup and code comments documenting which are exception events.

- **Single subscription:** Agent client creates exactly one connection per `subscribe_to_run()` call. Satisfies AC4 "does not poll the agent for status or responses". Disconnection mid-stream triggers error, not retry loop.

- **Pure pass-through architecture:** FastAPI is a dumb pipe: agent → FastAPI → client. No transformation, no delay, no status authority in normal flow. The agent owns the state machine; FastAPI only routes events and recovers from agent failures (timeout/disconnect).

- **Client disconnection handling:** Per comment 4374346590, SSE library detects `http.disconnect` and cancels task group, triggering agent subscription's finally block for cleanup. Implementation must verify task cancellation propagates correctly (integration test with real async runtime).

- **Implementation deferred:** Concrete file paths, imports, and language-specific syntax removed per ADLC rule: "File paths and stack MUST match repo context / existing tree. Never invent another language or framework." Implementation awaits repo-tree mapping.

- **Stack-agnostic design:** Operations defined as behavioral contracts, not language-specific code. Implementation adapts to repo's actual stack.

---

**Document Status:** Draft  
**Story:** 05-fastapi (PER-33)  
**Epic:** FastAPI SSE Integration with Agent Service (requirements.md AC2, AC3, AC4)  
**Altitude:** Story-level LLD
