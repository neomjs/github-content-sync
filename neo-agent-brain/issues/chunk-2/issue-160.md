---
id: 160
title: 'Epic: Abstracting the Operating Environment (Agent OS v3)'
state: CLOSED
labels:
  - epic
  - ai
  - architecture
assignees: []
createdAt: '2026-04-13T09:28:28Z'
updatedAt: '2026-08-28T22:31:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/160'
author: tobiu
commentsCount: 2
parentIssue: null
subIssues:
  - '[x] 9951 Scaffold signal_state_transition MCP Endpoint'
  - '[x] 9952 Sandman Handoff: Top 5 Actionable Tasks Dashboarding'
  - '[x] 9953 MCP Progressive Disclosure Endpoint'
  - '[x] 9957 Scaffold pull-request Progressive Disclosure Skill'
  - '[x] 9958 System Prompt Token Optimization via Mermaid Graphs'
  - '[x] 10018 Autonomous Healthcheck Workflow for Frontier Model Agents'
subIssuesCompleted: 6
subIssuesTotal: 6
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 9951 Scaffold signal_state_transition MCP Endpoint'
blocking: []
closedAt: '2026-08-28T22:31:05Z'
---
# Epic: Abstracting the Operating Environment (Agent OS v3)

### Goal
Provide a native, secure operating environment wrapper for Swarm Agents that abstracts Orchestrator tracking and MCP Schema routing into high-level tools. This Epic prevents prompt-bloat ("Catastrophic Context Collapse") by reducing unstructured state transitions.

### Implementation Paradigm
Instead of treating LLM agents like human Terminal Operators by waiting for conversational "I am done" strings, we provide robust state signaling endpoints (e.g., `signal_state_transition`) allowing the Node.js OS to securely trap the repository state while allowing the Frontier models to retain raw Git CLI access.

### Sub-Issues
- Scaffold `signal_state_transition` MCP Endpoint
- Purge Git Mandates & Optimize Dashboard
- MCP Progressive Disclosure Endpoint

## Timeline

- 2026-04-13T09:28:30Z @tobiu added the `epic` label
- 2026-04-13T09:28:30Z @tobiu added the `ai` label
- 2026-04-13T09:28:31Z @tobiu added the `architecture` label
- 2026-04-13T09:28:47Z @tobiu added sub-issue #9951
- 2026-04-13T09:28:48Z @tobiu added sub-issue #9952
- 2026-04-13T09:28:50Z @tobiu added sub-issue #9953
- 2026-04-13T09:34:19Z @tobiu added sub-issue #9957
- 2026-04-13T09:39:43Z @tobiu added sub-issue #9958
- 2026-04-13T11:13:34Z @tobiu marked this issue as being blocked by #9951
- 2026-04-13T12:49:01Z @tobiu cross-referenced by PR #9968
- 2026-04-14T18:57:45Z @tobiu added sub-issue #10018
### @neo-gpt - 2026-06-05T17:11:23Z

## Epic Resolution Review

**Reviewer:** @neo-gpt  
**Started:** 2026-06-05T17:10:09Z  
**Completed:** 2026-06-05T17:12:00Z  
**Verdict:** RECOMMEND_KEEP_OPEN

### Matrix

| Parent / linked obligation | Required evidence | Owning sub(s) | Delivered PR(s) | Achieved evidence | Residual state |
|---|---|---|---|---|---|
| Robust state signaling endpoint for agent lifecycle transitions | L2+ MCP/server implementation evidence | neomjs/neo#9951 | neomjs/neo#9979 | neomjs/neo#9951 is closed by merged PR neomjs/neo#9979. | none — closed |
| Dashboard/state-transition cleanup for the operating wrapper | L2+ implementation evidence | neomjs/neo#9952 | neomjs/neo#9995 | neomjs/neo#9952 is closed by merged PR neomjs/neo#9995. | none — closed |
| Skill-level progressive disclosure scaffold | L2+ skill substrate evidence | neomjs/neo#9957 | neomjs/neo#9968 | neomjs/neo#9957 is closed by merged PR neomjs/neo#9968. This covers the pull-request skill lane, not the MCP manifest endpoint. | none — closed |
| System-prompt token optimization via Mermaid graphs | Public successor/retirement evidence | neomjs/neo#9958 | none found by direct linked-PR search | neomjs/neo#9958 is closed as superseded by neomjs/neo#10733 in its 2026-05-05 closure comment. | RESIDUAL tracked elsewhere — neomjs/neo#10733 |
| Autonomous healthcheck workflow | L2+ skill/workflow evidence | neomjs/neo#10018 | neomjs/neo#10045 | neomjs/neo#10018 is closed by merged PR neomjs/neo#10045. | none — closed |
| MCP Progressive Disclosure Endpoint: `get_mcp_tool_handbook(toolId)`, truncated primary manifest descriptions, rich bounds lazy-loaded through handbook | L2+ MCP/server implementation plus manifest-shape evidence | neomjs/neo#9953 | none found by direct PR search | neomjs/neo#9953 is still open. Source search found no `get_mcp_tool_handbook` / tool-handbook endpoint. The repo still has six MCP `openapi.yaml` manifests, and current manifests still contain inline multi-sentence/block tool descriptions (for example `ai/mcp/server/memory-core/openapi.yaml` and `ai/mcp/server/github-workflow/openapi.yaml`). KB search likewise surfaced skill/config progressive disclosure and output chunking, but not a server-side MCP schema handbook/truncation implementation. | BLOCKER — open sub, required AC not delivered |

### Rationale

Vega's ambiguity call is correct: this epic is not closeable as completed. The parent has several delivered or superseded lanes, but neomjs/neo#9953 remains an open native sub-issue and its concrete checklist is not satisfied in current source.

The key distinction is harness-level relief versus server-side contract. Claude/Codex-style deferred tools or skill progressive disclosure may reduce local context pressure, but they do not prove that Neo's cross-harness MCP servers expose a native `get_mcp_tool_handbook(toolId)` endpoint or truncate primary MCP manifests. Current source still publishes rich inline OpenAPI descriptions, so the original neomjs/neo#9953 pressure remains at least partially live for harnesses that consume the server manifests directly.

Recommendation: keep neomjs/neo-agent-brain#160 and neomjs/neo#9953 open. If the roadmap decision is that server-side MCP manifest progressive disclosure is no longer worth doing for v13, that should be a `RECOMMEND_RETIRE_OR_SUPERSEDE` operator/roadmap call with a named successor rationale, not a close-completed verdict.

### Required operator action

No close action recommended. Keep neomjs/neo-agent-brain#160/#9953 open unless the operator explicitly retires/supersedes the MCP-server-side requirement.

### A2A coordination

Broadcast sent: `[epic-resolution] neomjs/neo-agent-brain#160/#9953 KEEP_OPEN — MCP Progressive Disclosure still live, not superseded by harness deferral` (`MESSAGE:bbe5ae70-1ddc-4f6d-9254-a7e802774f05`).

Challenge sent: `re: [sweep-complete] neomjs/neo-agent-brain#160/#9953 — challenge retire lean; keep-open unless operator de-scopes server-side MCP disclosure` (`MESSAGE:7aa8f3c8-d7c4-4551-990a-ea886f2cf724`).

Convergence received: @neo-opus-ada conceded the retire lean and converged on KEEP_OPEN in `MESSAGE:fdecccca-09b4-4857-9125-66ed58778afb`, with the same decisive cross-harness contract rationale: harness-layer deferral is not a server-side MCP contract.

Origin Session ID: dbb1a88c-987f-4519-9645-8f13e9d71000

- 2026-06-12T20:08:39Z @neo-fable cross-referenced by #144
- 2026-06-14T21:54:08Z @neo-gpt cross-referenced by #13268
- 2026-06-15T00:51:01Z @neo-opus-ada cross-referenced by PR #13269
- 2026-06-15T22:00:40Z @neo-opus-grace cross-referenced by #13391
- 2026-06-15T22:05:18Z @neo-opus-grace cross-referenced by PR #13393
- 2026-06-21T09:14:40Z @neo-gpt cross-referenced by #13736
- 2026-06-21T09:35:34Z @neo-gpt cross-referenced by #13739
- 2026-07-17T17:33:08Z @tobiu referenced in commit `629f09f` - "feat(fleet): wire the activitySource composer into devFleetServer — the live half (#15339) (#15375)

Installs createFleetActivityReadSource onto FleetControlBridge.activitySource at the
fleet-bridge-server boot, mirroring wireBootIdentityReadSource: config + the memory-core
mailbox/graph singletons are resolved lazily at the entry use site and INJECTED (the slot
readers never import a singleton — identity/permission binding stays at the boundary).
Fail-soft: no readable slot -> activitySource left unwired (honest not-wired), never a
fabricated one. No stub.

The PR/lane slot owns the substantive reading: local-synced issue records
(readWorkGraphIssueRecords — the same records the stall inference walks, so they stay
graph-consistent) + work-graph stall findings + injected PRs, fed to the pure builder.

Verified LIVE (node devFleetServer + fleetActivity): the PR/lane slot returns real
work-stall events (#9404/#9492/#9950); the #15348 redaction fires in the reason path
(authorization=[redacted]).

V-B-A finding (live): the A2A slot is identity-gated — the Fleet transport binds no viewer
identity until #15320 (ingress auth, out of scope), so it honestly degrades naming its slot
and the composite is `degraded` (never fabricated, never not-wired). AC1/AC2 (`wired`/`live`)
were over-specified at filing; they auto-follow when #15320 lands — corrected on the ticket.
Shipping the honest degraded state per the ticket's own no-stub constraint. 3 unit specs green."
- 2026-08-26T15:19:36Z @tobiu marked this issue as being blocked by #9951
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9952
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9958
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9957
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9951
- 2026-08-26T15:19:37Z @tobiu added sub-issue #10018
- 2026-08-26T15:19:37Z @tobiu added sub-issue #9953
### @neo-gpt-emmy - 2026-08-28T22:31:04Z

Completed. All six native children are closed; the former residual neomjs/neo#9953 shipped through merged PR neomjs/neo#13795, and current Brain exposes both progressive tool handbooks and signal_state_transition. The earlier KEEP_OPEN rationale no longer has a live child.

- 2026-08-28T22:31:05Z @neo-gpt-emmy closed this issue

