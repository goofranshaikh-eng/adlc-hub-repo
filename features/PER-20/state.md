---
schema_version: 1
key: PER-20
status: pr_reviewed
current_node: pr_review
track: standard
involved_repos:
- graphql
- mcp
- agent
depends_on: []
owner: goos
lease_until: '2026-10-23T10:09:19.354316+00:00'
revision: 795
hub_sha: null
ticket_hash: 69572336c037b7b7
jira_hash: 69572336c037b7b7
tpm_approved: true
specs_locked: true
skills_lock:
  bmad: 1cbcfa272fe65787c06a1fa164a901f46117cca7
stories:
- 01-agent
- 02-agent
- 03-agent
- 04-agent
child_ticket_keys:
- PER-21
- PER-22
- PER-23
- PER-24
critic_iterations: 0
dev_retries: 0
pr_revision_rounds: 6
pr_ids:
  hub_tpm: '7'
  hub_arch: '8'
  hub: '8'
  agent: '3'
waiting_for: feature_pr_human
tokens_in: 1472253
tokens_out: 199559
cost_anthropic_usd: 7.332786
history:
- node: intake
  note: snapshot written
  at: '2026-10-21T05:54:08.244144+00:00'
- node: triage
  note: cli override
  at: '2026-10-21T05:54:08.245240+00:00'
- node: analyst
  note: wrote requirements.md
  at: '2026-10-21T05:54:25.717657+00:00'
- node: analyst
  note: proposed_tickets=1
  at: '2026-10-21T05:54:32.687749+00:00'
- node: tpm_gate
  note: awaiting human hub PR approval (TPM)
  at: '2026-10-21T05:54:39.786773+00:00'
- node: tpm_gate
  note: approved via hub PR (reconcile)
  at: '2026-10-21T06:03:55.428778+00:00'
- node: tpm_gate
  note: "forge unlocked tpm_review \u2192 resume"
  at: '2026-10-21T06:03:55.444992+00:00'
- node: ticketize
  note: deferred Linear; proposed_tickets=1
  at: '2026-10-21T06:03:56.455568+00:00'
- node: ticketize
  note: deferred Linear; proposed_tickets=1
  at: '2026-10-21T06:10:43.079400+00:00'
- node: architect
  note: wrote hld.md
  at: '2026-10-21T06:11:50.519666+00:00'
- node: architect
  note: specs=['1']
  at: '2026-10-21T06:12:21.187688+00:00'
- node: scrum_master
  note: stories=['01-agent', '02-agent', '03-agent', '04-agent']; linear=['PER-21',
    'PER-22', 'PER-23', 'PER-24']
  at: '2026-10-21T06:13:46.712243+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T06:19:04.903690+00:00'
- node: test_architect
  note: tests appended
  at: '2026-10-21T06:23:05.390774+00:00'
- node: critic
  note: pass iter=1
  at: '2026-10-21T06:23:30.865629+00:00'
- node: merge_pack
  note: merged 4 ticket(s)
  at: '2026-10-21T06:23:30.872495+00:00'
- node: architect_gate
  note: awaiting human hub PR approval
  at: '2026-10-21T06:23:36.329914+00:00'
- node: architect_gate
  note: approved via hub PR (reconcile)
  at: '2026-10-21T06:44:36.591510+00:00'
- node: architect_gate
  note: "forge unlocked architect_review \u2192 resume"
  at: '2026-10-21T06:44:36.609787+00:00'
- node: dev
  note: starting
  at: '2026-10-21T06:44:37.660603+00:00'
- node: dev
  note: done 01-agent
  at: '2026-10-21T06:47:39.815423+00:00'
- node: dev
  note: done 02-agent
  at: '2026-10-21T06:49:11.209698+00:00'
- node: dev
  note: done 03-agent
  at: '2026-10-21T06:50:46.390563+00:00'
- node: dev
  note: done 04-agent
  at: '2026-10-21T06:53:38.083630+00:00'
- node: pre_pr_qa
  note: report written
  at: '2026-10-21T06:53:38.091275+00:00'
- node: open_pr
  note: push agent ok
  at: '2026-10-21T07:09:29.028413+00:00'
- node: open_pr
  note: PRs opened agent:3
  at: '2026-10-21T07:09:29.028479+00:00'
- node: pr_review
  note: diff loaded agent chars=26718
  at: '2026-10-21T07:09:30.164996+00:00'
- node: pr_review
  note: 'inline review agent: 10 comment(s)'
  at: '2026-10-21T07:10:13.664323+00:00'
- node: pr_review
  note: 'agent:issue:major:**AC-2 violation: Wrong MCP transport protocol.** The code
    sends a plain HTTP PO'
  at: '2026-10-21T07:10:18.577903+00:00'
- node: pr_review
  note: "major issue classified \u2192 human (deferred; continue auto-fix)"
  at: '2026-10-21T07:10:20.120264+00:00'
- node: pr_review
  note: agent:issue:major:**Missing FastMCP client dependency.** Requirements AC-2
    and HLD ADR-2 mandate F
  at: '2026-10-21T07:10:21.956622+00:00'
- node: pr_review
  note: "major issue classified \u2192 human (deferred; continue auto-fix)"
  at: '2026-10-21T07:10:23.956260+00:00'
- node: pr_review
  note: 'agent:issue:moderate:**AC-2 gap: Tool inference logic incomplete.** `_infer_tool`
    only handles `list_'
  at: '2026-10-21T07:10:25.889018+00:00'
- node: pr_review
  note: 'agent:issue:major:**AC-1/AC-2 risk: No actual LLM tool-use API.** The code
    calls Claude''s plain `m'
  at: '2026-10-21T07:10:30.151089+00:00'
- node: pr_review
  note: "major issue classified \u2192 human (deferred; continue auto-fix)"
  at: '2026-10-21T07:10:31.995191+00:00'
- node: pr_review
  note: 'agent:issue:moderate:**Test gap: AC-2 not verified.** Test mocks `httpx.post`
    and asserts JSON-RPC pa'
  at: '2026-10-21T07:10:34.047908+00:00'
- node: pr_review
  note: 'agent:issue:moderate:**Test gap: AC-2 tool coverage incomplete.** Test only
    covers `get_employee_by_i'
  at: '2026-10-21T07:10:38.350664+00:00'
- node: pr_review
  note: 'agent:needs_human:major:**AC-3 gap: No error surfacing on tool failure.**
    When `tool_result` contains `{'
  at: '2026-10-21T07:10:42.130896+00:00'
- node: pr_review
  note: 'agent:issue:major:**AC-5 process note: Example questions claim incorrect
    behavior.** README line 9'
  at: '2026-10-21T07:10:45.509615+00:00'
- node: pr_review
  note: "major issue classified \u2192 human (deferred; continue auto-fix)"
  at: '2026-10-21T07:10:47.098581+00:00'
- node: pr_review
  note: 'agent:issue:moderate:**AC-4 clarification: Health probe method mismatch.**
    Code uses `client.head(url'
  at: '2026-10-21T07:10:48.803647+00:00'
- node: pr_review
  note: 'agent:needs_human:moderate:**Exception criterion violation: ID normalization
    incomplete.** `_normalize_id` '
  at: '2026-10-21T07:10:52.677730+00:00'
- node: pr_review
  note: auto-fix tier=moderate round=1
  at: '2026-10-21T07:10:54.057639+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T07:10:54.529047+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T07:16:44.734547+00:00'
- node: dev
  note: starting
  at: '2026-10-21T07:16:44.734571+00:00'
- node: dev
  note: done 01-agent
  at: '2026-10-21T07:30:14.924282+00:00'
- node: dev
  note: done 02-agent
  at: '2026-10-21T07:34:44.352959+00:00'
- node: dev
  note: done 03-agent
  at: '2026-10-21T07:44:35.485674+00:00'
- node: dev
  note: done 04-agent
  at: '2026-10-21T07:48:12.547965+00:00'
- node: open_pr
  note: push agent ok
  at: '2026-10-21T07:48:14.899613+00:00'
- node: pr_review
  note: still_open agent:4333988092
  at: '2026-10-21T07:48:21.188797+00:00'
- node: pr_review
  note: resolved agent:4333988117
  at: '2026-10-21T07:48:27.645812+00:00'
- node: pr_review
  note: still_open agent:4333988123
  at: '2026-10-21T07:48:32.479552+00:00'
- node: pr_review
  note: resolved agent:4333988155
  at: '2026-10-21T07:48:37.704723+00:00'
- node: pr_review
  note: "major issue \u2192 delta spec gate (human) after minor/moderate auto-fix"
  at: '2026-10-21T07:48:37.706597+00:00'
- node: pr_review
  note: 'fix:agent:needs_human:major:**AC-3 gap: No error surfacing on tool failure.**
    When `tool_result` contains `{'
  at: '2026-10-21T08:08:00.580118+00:00'
- node: pr_review
  note: fix code agent:4333988134 tier=moderate
  at: '2026-10-21T08:08:07.084225+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T08:08:07.707612+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T08:13:56.612910+00:00'
- node: dev
  note: starting
  at: '2026-10-21T08:13:56.613230+00:00'
- node: dev
  note: changes requested
  at: '2026-10-21T08:34:04.416275+00:00'
- node: architect_gate
  note: approved via CLI
  at: '2026-10-21T08:36:36.621117+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T08:36:37.461415+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T08:42:10.835591+00:00'
- node: dev
  note: starting
  at: '2026-10-21T08:42:10.835898+00:00'
- node: open_pr
  note: push agent ok
  at: '2026-10-21T09:03:39.500374+00:00'
- node: pr_review
  note: still_open agent:4333988134
  at: '2026-10-21T09:03:45.765593+00:00'
- node: pr_review
  note: resolved agent:4333988165
  at: '2026-10-21T09:03:52.351169+00:00'
- node: pr_review
  note: review fix done handled=4
  at: '2026-10-21T09:03:52.351994+00:00'
- node: catch_up
  note: "unrepaired feature PR comments \u2192 fix"
  at: '2026-10-21T09:04:58.560847+00:00'
- node: pr_review
  note: 'fix:agent:issue:moderate:ADLC decision: action=fix_code tier=moderate (Tool
    inference logic incomplete fo'
  at: '2026-10-21T09:05:02.846920+00:00'
- node: pr_review
  note: fix code agent:4333992871 tier=moderate
  at: '2026-10-21T09:05:04.265303+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T09:05:04.791076+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T09:10:21.143651+00:00'
- node: dev
  note: starting
  at: '2026-10-21T09:10:21.145015+00:00'
- node: pr_review
  note: 'fix:agent:issue:moderate:ADLC decision: action=fix_code tier=moderate (Test
    lacks verification of AC-2 (F'
  at: '2026-10-21T09:36:19.236171+00:00'
- node: pr_review
  note: fix code agent:4334244461 tier=moderate
  at: '2026-10-21T09:36:21.076594+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T09:36:21.706335+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T09:42:26.133947+00:00'
- node: dev
  note: starting
  at: '2026-10-21T09:42:26.134154+00:00'
- node: pr_review
  note: 'fix:agent:needs_human:moderate:ADLC: still open (verify did not confirm fix)'
  at: '2026-10-21T10:07:45.571060+00:00'
- node: pr_review
  note: fix dismissed agent:4334286480
  at: '2026-10-21T10:07:49.756444+00:00'
- node: pr_review
  note: 'fix:agent:not_an_issue:None:ADLC: resolved after verify'
  at: '2026-10-21T10:07:51.309816+00:00'
- node: pr_review
  note: 'fix:agent:needs_human:moderate:ADLC: still open (verify did not confirm fix)'
  at: '2026-10-21T10:07:55.339364+00:00'
- node: pr_review
  note: fix dismissed agent:4334287948
  at: '2026-10-21T10:07:59.884611+00:00'
- node: open_pr
  note: push agent ok
  at: '2026-10-21T10:08:02.348205+00:00'
- node: pr_review
  note: still_open agent:4333992871
  at: '2026-10-21T10:08:07.731016+00:00'
- node: pr_review
  note: still_open agent:4334244461
  at: '2026-10-21T10:08:12.592241+00:00'
- node: pr_review
  note: review fix done handled=5
  at: '2026-10-21T10:08:12.605066+00:00'
- node: catch_up
  note: "unrepaired feature PR comments \u2192 fix"
  at: '2026-10-21T10:09:19.354415+00:00'
- node: pr_review
  note: 'fix:agent:not_an_issue:None:ADLC: resolved after verify'
  at: '2026-10-21T10:09:23.087072+00:00'
- node: pr_review
  note: 'fix:agent:not_an_issue:None:ADLC decision: action=fix_code tier=moderate
    (fallback: fix_code).'
  at: '2026-10-21T10:09:26.326964+00:00'
- node: pr_review
  note: 'fix:agent:needs_human:major:ADLC decision: action=update_specs_then_code
    tier=moderate (Exception criterion '
  at: '2026-10-21T10:09:30.199682+00:00'
- node: pr_review
  note: fix code agent:4334675272 tier=moderate
  at: '2026-10-21T10:09:36.346418+00:00'
- node: pr_review
  note: "route moderate \u2192 designer then dev"
  at: '2026-10-21T10:09:37.131611+00:00'
- node: designer
  note: lld updated
  at: '2026-10-21T10:16:23.796488+00:00'
- node: dev
  note: starting
  at: '2026-10-21T10:16:23.796520+00:00'
- node: pr_review
  note: 'fix:agent:needs_human:moderate:ADLC: still open (verify did not confirm fix)'
  at: '2026-10-21T10:45:41.107426+00:00'
- node: pr_review
  note: fix dismissed agent:4334940231
  at: '2026-10-21T10:45:44.970171+00:00'
- node: open_pr
  note: push agent ok
  at: '2026-10-21T10:45:49.628561+00:00'
- node: pr_review
  note: still_open agent:4334675272
  at: '2026-10-21T10:45:54.763807+00:00'
- node: pr_review
  note: review fix done handled=5
  at: '2026-10-21T10:45:54.764114+00:00'
---

# Feature state

See frontmatter for machine-readable fields.
