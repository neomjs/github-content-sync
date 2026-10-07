---
id: 911
title: Make Stop cancel pending managed Starts
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-07T01:15:42Z'
updatedAt: '2026-10-07T01:15:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/911'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 909 Launch Desktop MCPs through scoped Fleet admission'
blocking: []
---
# Make Stop cancel pending managed Starts

## Context
While repairing PR `#910`'s early-Stop admission arm, the production managed-Start path showed a separate lifecycle gap: Stop can arrive during asynchronous preparation, return a stopped result with no process record, and the pending Start can later spawn the harness.

Design authority: [the planner disposition under `#571`](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6028624472) explicitly adds logical cancellation of pending managed Starts: “A canceled attempt must settle without a new spawn or wake arming; a later explicit Start is a fresh attempt.” This is an enhancement of the currently documented process-stop behavior; it does not claim that `#909` already owns general launch cancellation.

## The Problem
At `#910` head `edc0c7eafe7c9a770418d01281bed5d4185661b1`, the admission revocation mark prevents a later Claude Desktop MCP redemption, but the harness process still starts. The base `f5ee2bcf` has the same general gap. The path is shared across managed harness families, so restricting the repair to Desktop admission leaves the operator's Stop intent incomplete.

This is source-derived evidence, supported by the production-composer regression described in [the author response](https://github.com/neomjs/neo-agent-brain/pull/910#issuecomment-6028423503). No live seat was started or stopped for this report.

## The Architectural Reality
- [FleetManager.startAgent](https://github.com/neomjs/neo-agent-brain/blob/edc0c7eafe7c9a770418d01281bed5d4185661b1/ai/services/fleet/FleetManager.mjs#L406) captures a mark before `withSeatHome`, but uses it for launch admission.
- [spawnPermitted](https://github.com/neomjs/neo-agent-brain/blob/edc0c7eafe7c9a770418d01281bed5d4185661b1/ai/services/fleet/startAgentProvisioned.mjs#L75) rechecks launch ownership and participation, then calls the lifecycle start.
- [FleetLifecycleService.stop](https://github.com/neomjs/neo-agent-brain/blob/edc0c7eafe7c9a770418d01281bed5d4185661b1/ai/services/fleet/FleetLifecycleService.mjs#L979) has no pending process to stop before spawn. Its early return does not currently prevent that pending launch.

## The Fix
Give pending managed Starts one lifecycle-owned cancellation boundary. Capture the attempt before asynchronous preparation or seat-home waiting; retain Stop against the already-pending attempts; check cancellation through to the actual spawn boundary, including asynchronous work inside lifecycle start. An explicit later Start gets a fresh attempt.

Reuse/generalize the existing fencing mechanism where appropriate, with one authority for Stop cancellation. MCP admission must consume that same cancellation fact; do not maintain independently advancing Stop counters per harness or in parallel lifecycle/admission owners. Retire or redirect any superseded admission-only tracking rather than layering another scheduler or queue beside it.

A phase already in progress may finish under its existing bounds, and prepared repository files may remain. Cancellation must prevent subsequent spawn, lease activation and wake arming from that attempt. If the process already exists, preserve the existing stop/finalization path.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Pending `FleetManager.startAgent` / managed composer | Planner disposition above; existing seat-home serialization | Stop fences all attempts already pending for that seat, including queued preparation | Later explicit Start remains eligible; other seats unaffected | Start/Stop JSDoc | Paused preparation and queued-Start controls over production functions |
| `FleetLifecycleService.stop` and pre-spawn boundary | Same disposition; existing process supervisor | Retain cancellation even without a process record; no late spawn after Stop | Already-spawned process follows current cleanup; no-pending Stop keeps its existing no-process behavior | Existing finite result shape and documented cancellation reason | Early/late race, no-child and running-process controls |
| MCP admission cancellation | `#909`'s accepted contract | Uses the same Stop cancellation fact and cannot activate a canceled attempt | Independent non-Stop grant revocations retain their existing semantics | Admission/lifecycle ownership docs | Existing `#910` regressions plus later-Start control |

Decision Record impact: aligned with Fleet-owned lifecycle and the existing admission boundary. No new credential audience, service, configuration leaf or wire verb.

## Acceptance Criteria
- [ ] Stop during queued seat-home work, checkout or workspace preparation prevents the affected attempts from reaching spawn and settles them as canceled rather than successful starts.
- [ ] Stop during asynchronous lifecycle preparation is checked at the final spawn boundary; the guard is not merely before another awaited function.
- [ ] A canceled attempt performs no later lease/admission activation or wake arming. Already completed repository preparation need not be rolled back.
- [ ] A later explicit Start, including the fresh Start of a restart, can proceed; old canceled continuations cannot cancel or overwrite its state.
- [ ] The behavior holds through the shared managed path for Desktop and a non-Desktop harness, with cross-seat and no-pending-Start negative controls.
- [ ] Running/adopted process stop and existing profile-helper finalization retain their current guarantees.
- [ ] One owner carries the cancellation fact; existing admission tests remain green without a second independently advancing Stop epoch.
- [ ] The finite Stop/Start results and JSDoc distinguish a canceled pending attempt from stopping an existing process. Production-function tests pause real asynchronous seams with synthetic dependencies; no live harness is needed.

## Out of Scope
New UI controls, global plane shutdown redesign, forcibly killing unrelated provisioning processes, rolling back prepared repositories, and new permission/credential work. `#910` must still satisfy its own admission action independently.

## Related and sequencing
Parent: #571. Follow the shared source seam after #909; this is not an added merge prerequisite for PR #910.
BLOCKED_BY #909

## Filing evidence
The current-head/base trace and independent read agree. Structure-map command exited 0; existing owner is `ai/services/fleet`, with no new file/directory prescribed. MC problem queries recovered original lifecycle work (memory `3032dbc7-db37-4728-a0d5-4c3316ee38c1`) but no prior cancellation decision; own-assignment sweep found five Brain issues, none on this contract. Live latest-open sweep: latest 20 open issues checked at 2026-10-07 01:15:19 UTC; no equivalent found. Recent all-status A2A claim sweep found no competing author. MC problem queries support the source/consumer separation; no superseding prior decision surfaced.

Origin Session ID: d0d0bed3-7ce4-4bce-a16d-59589484aec0
Retrieval Hint: Stop during pending managed Start; withSeatHome; spawnPermitted; PR910 early-Stop admission versus process spawn.


## Timeline

- 2026-10-07T01:15:42Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-07T01:15:43Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-07T01:15:43Z @neo-gpt-emmy added the `ai` label
- 2026-10-07T01:15:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-07T01:16:00Z @neo-gpt-emmy added parent issue #571
- 2026-10-07T01:16:01Z @neo-gpt-emmy marked this issue as being blocked by #909
- 2026-10-07T01:17:07Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-07T01:17:08Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-07T01:21:34Z @neo-gpt cross-referenced by #477

