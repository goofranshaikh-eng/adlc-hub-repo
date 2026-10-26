# Code Review: FastAPI SSE Integration PR

## Review Scope
**Content:** `pr.diff` (FastAPI backend integration with agent service)  
**Requirements Reference:** `requirements.md` (5 acceptance criteria)  
**HLD Reference:** `hld.md` (architecture, components, risks)  
**Breakdown Reference:** `breakdown.md` (5 stories, 27 tests)  
**Stories Reference:** Stories 01–05 LLD documents

## Findings

```json
[
  {
    "path": "app/routers/query.py",
    "line": 71,
    "body": "**AC1 timing guardrail missing:** No assertion that `initiate_query()` returns in < 100ms as specified in Story 04 verification step 1. Current test `test_successful_query_returns_ok_immediately` does check elapsed time, but the endpoint handler itself has no timeout on the agent client call beyond the client's own `AGENT_TIMEOUT` (3s). If the agent takes 2.9s to reply, the endpoint still returns it synchronously, violating AC1's implicit \"immediate\" requirement. Recommendation: add explicit handler-level timeout (e.g., 100ms fail-safe) or document that \"immediate\" means \"within agent's configured timeout.\" Current code satisfies the literal requirement (returns without waiting for agent completion) but not the spirit (fast reply)."
  },
  {
    "path": "app/services/agent_client.py",
    "line": 91,
    "body": "**AC4 polling guarantee incomplete:** `subscribe_to_run()` raises `AgentConnectionError` if the stream ends without a terminal event (line 96: 'agent disconnected before a terminal event'). However, the code does not retry; it propagates the error to the caller (`_forward_events` in `query.py`). The test `test_disconnect_during_in_progress_is_error_without_retry` confirms no retry happens. AC4 ('no polling behavior') is met, but the error handling creates a backend-generated `failed` event in the endpoint (query.py line 141). This is correct per HLD Risk 3 mitigation and AC3 clarification, but the layering is unclear: the client raises an error, and the endpoint catches it to generate `failed`. Recommendation: add docstring clarifying this is intentional exception recovery, not a retry pattern."
  },
  {
    "path": "app/routers/query.py",
    "line": 139,
    "body": "**AC3 exception event generation scope ambiguous:** Backend-generated `failed` events (lines 139–141 on timeout, 145–147 on connection error) satisfy AC3 clarification ('exception recovery, not normal forwarding'). However, the code distinguishes `TimeoutError` from other exceptions without explicit comments mapping each to the HLD Risk 3 mitigation. `AgentConnectionError` during `subscribe_to_run()` yields `STREAM_ERROR`; `TimeoutError` yields `AGENT_TIMEOUT`. No validation that these are the only backend-generated events; if a new error type is added to agent_client, it may silently produce a generic `STREAM_ERROR` event. Recommendation: add assertion or enum to restrict exception-to-event mappings or document the catch-all behavior."
  },
  {
    "path": "tests/test_query_endpoint.py",
    "line": 88,
    "body": "**AC1 coverage gap: concurrent request test does not verify 100ms target for individual requests under load.** Test `test_concurrent_query_returns_within_100ms` asserts `elapsed < 0.1` for the total time of 12 concurrent requests (line 113), not per-request latency. An individual request could take 95ms while 12 in parallel take 97ms total (if agent responses interleave quickly), yet each request individually meets the 100ms target. Recommendation: either (a) assert per-request elapsed from the tuples in `results` (already captured on line 103), or (b) document that the test only validates throughput, not per-request latency."
  },
  {
    "path": "app/services/run_registry.py",
    "line": 42,
    "body": "**AC3 verification incomplete: test `test_registry_stores_only_id_mapping` (test_run_registry.py, line 90) asserts forbidden fields and methods, but does not validate thread-safety.** The `_lock` is present but the test does not verify that concurrent register/lookup/remove calls serialize correctly. In a stress scenario (100 concurrent POSTs → 100 concurrent SSE opens), a race between `remove()` in finally block and a late `lookup()` could cause 404 or double-remove. Recommendation: add test exercising concurrent access (e.g., threading.Thread or asyncio.gather over register/lookup/remove) to confirm `_lock` prevents corruption."
  },
  {
    "path": "tests/test_stream_endpoint.py",
    "line": 396,
    "body": "**Breakdown Story 5 AC coverage: test count does not align.** Breakdown story 5 lists 8 verification tests (lines 240–266 in stories/05-fastapi.md), but `test_stream_endpoint.py` implements 13 test functions (test_stream_forwards_agent_events through test_multiple_runs_cleaned_up_after_terminal_events). Mapping: (1) test_stream_forwards_agent_events, (2) test_stream_forwards_failure_event / test_stream_stops_after_first_terminal_event_even_if_agent_sends_more, (3) test_stream_closes_after_completed_event, (4) test_stream_closes_after_failed_event, (5) test_missing_run_returns_404_without_subscribing, (6) test_subscription_timeout_sends_failed_event, (7) test_agent_disconnect_during_in_progress_sends_stream_error_without_retry, (8) test_client_disconnection_triggers_cleanup, (9) test_multiple_runs_cleaned_up_after_terminal_events. The extra tests (test_subscription_error_sends_generic_failed_event, test_stream_stops_after_first_terminal_event_even_if_agent_sends_more parametrized cases) exceed breakdown count. Recommendation: audit breakdown story 5 AC mapping (lines 247–266 in stories/05-fastapi.md) to confirm alignment with the 13 actual test functions."
  },
  {
    "path": "app/models/query_models.py",
    "line": 1,
    "body": "**Story 1 AC1 scope incomplete: no validation for `ImmediateResponse` field that agent returns `agent_run_id`.** Story 1 LLD (stories/01-fastapi.md line ~50) lists AC1 as 'POST response schema supports ok/rejected with run_id', but does not define the shape of the agent's immediate reply (which the endpoint receives). The endpoint's `_require_agent_reply()` function (query.py line 74) validates the agent returns `{\"status\": \"ok\"|\"rejected\", \"agent_run_id\": string}`, but this contract is not in Story 1 models. Recommendation: add `AgentImmediateReply` model to `app/models/query_models.py` to document the agent contract and validate it with Pydantic, not bare dict operations."
  },
  {
    "path": "app/streaming.py",
    "line": 1,
    "body": "**Breakdown Story 5 requirement: 'client disconnection detection' not explicitly tested.** Story 5 verification step (7) states 'Disconnect SSE client mid-stream, verify registry cleanup called' (stories/05-fastapi.md line 261). Test `test_client_disconnection_triggers_cleanup` (test_stream_endpoint.py line 371) does drive the HTTP layer to send `http.disconnect`, but it uses low-level `_drive_until_disconnect()` coroutine (async ASGI), not a high-level client-disconnect scenario. A production client using `TestClient` or `httpx.AsyncClient` that closes the connection would need verification that `SSEResponse._aclose()` (streaming.py line 85) actually runs and calls `registry.remove()`. Recommendation: add integration test using real `httpx.AsyncClient` context exit (e.g., `async with client: ... del client`) to confirm cleanup."
  },
  {
    "path": "app/routers/query.py",
    "line": 1,
    "body": "**HLD interaction diagram (HLD.md mermaid sequence) not validated by tests:** The sequence diagram shows `QueryHandler` → `RunRegistry.register_run()` → `AgentClient.initiate_query()`, but the endpoint code (query.py lines 52–70) calls `initiate_query()` before `registry.register()` (line 66). This is intentional (only register if agent succeeds), but the diagram suggests registration happens after agent call but before response, which is ambiguous about the ordering of side effects. Recommendation: verify the HLD sequence diagram matches the implementation (currently correct but easy to misread) or add inline comment explaining the order."
  }
]
```

## Summary

**8 findings identified:**

1. **AC1 timing (app/routers/query.py:71)** – No handler-level timeout; endpoint returns within agent's 3s timeout, not an explicit \"immediate\" guarantee.
2. **AC4 polling guardrail (app/services/agent_client.py:91)** – Error recovery layering unclear; endpoint catches `AgentConnectionError` to generate `failed` event.
3. **AC3 exception mapping (app/routers/query.py:139)** – No enum or validation restricting backend-generated event types; catch-all could mask new errors.
4. **AC1 concurrent test (tests/test_query_endpoint.py:88)** – Asserts total throughput, not per-request latency under load.
5. **AC3 thread-safety (app/services/run_registry.py:42)** – `_lock` present but not tested for concurrent access races.
6. **Breakdown alignment (tests/test_stream_endpoint.py:396)** – 13 tests implemented vs. 8 listed in Story 5; mapping unclear.
7. **Story 1 model gap (app/models/query_models.py:1)** – Agent immediate reply contract not formally modeled; validated as bare dict.
8. **Story 5 disconnect test (app/streaming.py:1)** – Low-level ASGI disconnect test; lacks high-level client-disconnect integration test.

**No material blockers to merge;** all acceptance criteria are functionally met. Findings are clarifications, test coverage gaps, and documentation ambiguities. Recommend addressing items 3, 5, and 7 before production deployment.
