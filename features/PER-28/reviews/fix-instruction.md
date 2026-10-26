Fix these forge PR review comments (auto-route):

### fastapi comment 4374346571
**AC1 Verification Gap**: Endpoint returns `ImmediateResponse` with client-generated `run_id`, but per AC1 and HLD, the immediate reply should be the agent's synchronous response without delay. The endpoint correctly does not wait for agent completion, but the test suite should explicitly verify sub-100ms response time under load to confirm AC1 is met (see Story 4 breakdown, Test 1: 'verify POST /query returns...in < 100ms').

### fastapi comment 4374346572
**AC4 Verification Gap**: `subscribe_to_run()` correctly creates a single SSE subscription and breaks after terminal event, matching AC4 'no polling behavior'. However, the code should explicitly document that agent disconnection during a non-terminal event is treated as an error (per HLD Risk 3), not silently retried. Test `test_single_connection_per_subscription()` verifies only one request is sent; edge case of disconnection mid-stream during in_progress is not covered by the current test suite.

### fastapi comment 4374346576
**AC2/AC3 Inconsistency**: Test `test_subscription_timeout_sends_failed_event()` verifies that a timeout generates a `failed` event with 'Agent timeout' message. Per AC3 ('FastAPI does not create...run status'), this event is agent-sourced in the normal flow, but when the agent times out, the backend *does* create and send a `failed` event. This is intentional per HLD (Risk 3 mitigation), but the test comment should clarify that this is a backend-generated exception event, not a forwarded agent event, to distinguish it from AC3's no-creation rule.

### fastapi comment 4374346581
**AC1 Implementation Check**: Per AC1 and Story 4 requirements, the endpoint must return the agent's immediate reply *without delay*. The code path is synchronous (no `await` between agent reply and return), which is correct. However, per HLD AD-4, the code should ensure exactly one terminal event (completed XOR failed) is guaranteed. The current SSE forwarding logic (line 118) breaks after any terminal event, which is correct, but this guarantee is not tested for a malformed agent stream that sends both `completed` and `failed`.

### fastapi comment 4374346586
**AC2/AC4 Coverage Gap**: The test suite covers forwarding of agent events (AC2) and absence of polling (AC4) for the happy path and timeout cases. However, per HLD AD-4 ('After forwarding `completed` or `failed`, stream closes; never send both'), no test verifies that a malicious or malformed agent stream sending `completed` followed by another event is correctly truncated. Add test: 'test_stream_stops_after_first_terminal_event_even_if_agent_sends_more'.

### fastapi comment 4374346590
**AC2 Implementation Detail**: The `SSEResponse` class correctly detects client disconnection via `http.disconnect` and cancels the task group, which triggers the agent subscription's finally block for cleanup (per HLD Risk 5). However, per Story 5 requirements ('cancel agent subscription on disconnection; Do NOT cancel agent run itself'), the code does not cancel the agent run—it only closes the local subscription. This is correct per spec, but the comment should be explicit: cleanup removes the local registry entry but the agent's background job continues.

### fastapi comment 4374346608
**AC3/AC4 Pass-Through Design**: The `_forward_events()` generator correctly forwards events exactly as received from the agent (line 119: `yield _sse_frame(event_type, event_data)`), with no modification, delay, or timer logic. Per AC3, this is correct. However, per HLD AD-3 ('all in_progress, completed, failed events originate from agent; backend only forwards'), if the agent sends an event with an unexpected type (not in `_TERMINAL_EVENTS`), the stream continues indefinitely. Add a guard: 'if agent sends event after terminal, log warning and break' to prevent infinite forwarding of non-terminal spam.
