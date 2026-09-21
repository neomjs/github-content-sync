---
id: 311
title: inspect_component_render_tree `both` sends a method name the engine never answers
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-04T17:56:54Z'
updatedAt: '2026-09-04T23:02:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/311'
author: neo-fable
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T23:02:59Z'
---
# inspect_component_render_tree `both` sends a method name the engine never answers

## Context

Found while building the engine's wire-name completeness spec for `neomjs/neo#18281`: a census of every `ConnectionService.call` the Brain's Neural Link services make against the engine client showed one reachable name the engine has never dispatched.

## The Problem

`ComponentService#inspectComponentRenderTree` (`ai/services/neural-link/ComponentService.mjs:93–110`) maps `type: 'both'` to the wire method `get_vdom_and_vnode`. The engine client registers `get_vdom_vnode` and implements `getVdomVnode` (engine `src/ai/client/ComponentService.mjs`, since engine commit `800a41fb45`); its dispatcher resolves the camelCase of the full name on the first matching prefix (`get_vdom`), finds no `getVdomAndVnode`, and answers `Unknown method: get_vdom_and_vnode`. The `type` enum in `ai/mcp/server/neural-link/openapi.yaml` (`:523`) advertises `both` and `ai/mcp/server/neural-link/toolService.mjs:54` routes the tool to this method, so the case is reachable and broken. The string predates the Brain split (present at `11552b0`) and the engine snapshot this repo pins carries the same pair, so the `both` case has been broken for as long as the two names have coexisted.

Verified 2026-09-04 by reading both sides at brain `ea336dd` and engine `dev@9fd9a8fdcc`; the engine-side spec `test/playwright/unit/ai/client/resolveServiceMethod.spec.mjs` (landing with `neomjs/neo#18281`) pins `get_vdom_vnode` as the engine's name for the `both` case.

## The Architectural Reality

The engine is the authority for the wire names it answers: each client service's `register*ServiceMethods` export lists them and the spec above resolves each one. The Brain's services are the senders, and a sender-side string has no engine witness — which is exactly how this drifted silently.

## The Fix

One line: the `both` branch sends `get_vdom_vnode`. Plus a unit arm on this side pinning the wire name per `type` against a recording `ConnectionService.call` stub, the `both` arm red first.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `inspect_component_render_tree` `type: 'both'` (`ComponentService.mjs:104`) | engine `registerComponentServiceMethods` + `ComponentService#getVdomVnode` | sends `get_vdom_vnode`, receives `{vdom, vnode}` | `vdom` / `vnode` cases unchanged | the tool description already promises "or both" | a live call with `type: 'both'` returns both trees |

## Decision Record impact

none.

## Acceptance Criteria

- [ ] `inspect_component_render_tree` with `type: 'both'` returns `{vdom, vnode}` from a live session (today: `Unknown method: get_vdom_and_vnode`).
- [ ] Unit, red-first: the arm asserting the wire name for `type: 'both'` fails on the current string and passes after the change.
- [ ] The `vdom` and `vnode` cases are unchanged.

## Out of Scope

- The two unrouted wrappers in the same file, `getComponentProperty` (`:44`) and `setComponentProperty` (`:146`): no operationId, no caller, no engine handler. Dead code, named here so the next reader does not take them for a second instance of this defect; removing them is a separate hygiene cut.

## Related

- `neomjs/neo#18281` — the engine-side wire-name spec that surfaced this.

unowned-rationale: a one-line sender fix plus one arm, claimable by any seat; filed from the engine lane that found it rather than left as a defect-note, because the tool's `both` case is advertised and reachable.

Live latest-open sweep: checked the latest 20 open issues (created-desc) at 2026-09-04T17:55:33Z; no equivalent found. A2A claim sweep (last 30 messages, all read-states): none on this surface. Memory Core sweep: implementation-era memories only (the `depth` parameter, `query_vdom`), no prior decision on this name pair; the string appears in the archived v11.18 build tickets as the original implementation, not as a known mismatch. Own-assignment sweep: seven open Brain tickets of mine, none on this surface. Structure map: `ai/services/neural-link` is the owning folder; the change stays in the service file that sends the name.

Origin Session ID: 8695dfb8-49e9-4368-a57d-67f5783808f8 (the lane turn; its recovery turn is 328f5750-f251-443f-8110-05d0b6dc9a38)

Retrieval Hint: `query_raw_memories("get_vdom_and_vnode get_vdom_vnode inspect_component_render_tree both Unknown method")`


## Timeline

- 2026-09-04T17:56:56Z @neo-fable added the `bug` label
- 2026-09-04T17:56:56Z @neo-fable added the `ai` label
- 2026-09-04T17:56:56Z @neo-fable added the `agent-os` label
- 2026-09-04T19:11:02Z @neo-gpt-emmy cross-referenced by PR #18300
- 2026-09-04T21:53:47Z @neo-fable assigned to @neo-fable
- 2026-09-04T22:01:07Z @neo-fable cross-referenced by PR #316
- 2026-09-04T23:02:59Z @tobiu referenced in commit `5c68018` - "Merge pull request #316 from neomjs/agent/311-both-wire-name

fix(neural-link): inspect_component_render_tree `both` sends the wire name the engine registers (#311)"
- 2026-09-04T23:02:59Z @tobiu closed this issue

