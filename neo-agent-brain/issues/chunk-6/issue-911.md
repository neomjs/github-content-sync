---
id: 911
title: Make Stop cancel pending managed Starts
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-07T01:15:42Z'
updatedAt: '2026-10-08T10:38:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/911'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 909 Launch Desktop MCPs through scoped Fleet admission'
blocking: []
closedAt: '2026-10-08T10:38:27Z'
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
Give pending managed Starts one lifecycle-owned cancellation boundary. Capture the attempt before asynchronous preparation or seat-home waiting; retain Stop against the already-pending attempts; check cancellation through to the actual spawn boundary, after awaited preparation and immediately before the lifecycle's actual spawn call. The current lifecycle `start` method is synchronous; preserve that shape rather than adding an artificial asynchronous layer. An explicit later Start gets a fresh attempt.

Reuse/generalize the existing fencing mechanism where appropriate, with one authority for Stop cancellation. MCP admission must consume that same cancellation fact; do not maintain independently advancing Stop counters per harness or in parallel lifecycle/admission owners. Retire or redirect any superseded admission-only tracking rather than layering another scheduler or queue beside it.

A phase already in progress may finish under its existing bounds, and prepared repository files may remain. Cancellation must prevent subsequent spawn, lease activation and wake arming from that attempt. If the process already exists, preserve the existing stop/finalization path.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Pending `FleetManager.startAgent` / managed composer | Planner disposition above; existing seat-home serialization | Stop fences all attempts already pending for that seat, including queued preparation | Later explicit Start remains eligible; other seats unaffected | Start/Stop JSDoc | Paused preparation and queued-Start controls over production functions |
| `FleetLifecycleService.stop` and pre-spawn boundary | Same disposition; existing process supervisor | Retain cancellation even without a process record; no late spawn after Stop | Already-spawned process follows current cleanup; no-pending Stop keeps its existing no-process behavior | Existing finite result shape and documented cancellation reason | Early/late race, no-child and running-process controls |
| MCP admission cancellation | `#909`'s accepted contract | Uses the same Stop cancellation fact and cannot activate a canceled attempt | Independent non-Stop grant revocations retain their existing semantics | Admission/lifecycle ownership docs | Existing `#910` regressions plus later-Start control |
| `FleetLifecycleService.beginStart` / `finishStart` / `canceledStart` | Lifecycle-owned pending attempt; manager captures before the seat-home queue | One AbortSignal shared through preparation, admission and wake arming; exact-attempt release and a finite canceled Start result | Never overwrite a newer process record; no-pending Stop keeps its prior response | Method JSDoc | Production-composer, queued-Start, final-spawn and stale-cleanup controls |
| `armFleetSeatWake` canceled-attempt receipt | Same lifecycle signal while the manager holds the seat home | Stop prevents later subscription/publication; a completed late subscription is withdrawn and the manifest reconciled before a newer Start | An unconfirmed withdrawal returns `unarmed`, `cleanupUnresolved: true`, and the known subscription ID; never claim removed or ready | Arming JSDoc | Proof/subscribe/publish cancellation and failed-withdrawal controls |

Decision Record impact: aligned with Fleet-owned lifecycle and the existing admission boundary. No new credential audience, service, configuration leaf or wire verb.

## Acceptance Criteria
- [ ] Stop during queued seat-home work, checkout or workspace preparation prevents the affected attempts from reaching spawn and settles them as canceled rather than successful starts.
- [ ] Stop during awaited capability or workspace preparation is checked again at the final lifecycle spawn boundary; the guard is not merely before another awaited function.
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
### @neo-gpt-emmy - 2026-10-08T01:23:32Z

## Intake — valid, with the current synchronous spawn boundary clarified

The live premise still holds at `dev@197e659a`: the manager captures an admission-only mark before the seat-home queue, but Stop with no process record does not cancel the composer. `spawnPermitted` rechecks ownership/participation, not that Stop. No competing open Brain PR covers this; #909 is closed. #925 remains a separate trust-projection PR.

**Prescription checked:** `ai/services/fleet/FleetLifecycleService.mjs` owns process Start/Stop and therefore the pending-attempt cancellation fact. The manager captures it before queueing, the composer carries it through preparation, and admission/wake arming consume it. Keep one Stop owner; preserve non-Stop admission revocations. No new operator input or UI action is introduced. An independent local source read agrees with this ownership. The current `start()` is synchronous, so I corrected my body’s asynchronous wording; the guard still belongs immediately before spawn, after upstream awaited preparation.

The parent has [Euclid’s independent epic review](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5931143185). Prior-art MC `0003be88`, `5331c990`, and `26e556a3` recover the planner split and #910’s admission-only scope; current source is the falsifier. `pre_brief_session` did not resolve this qualified issue ID, so those targeted records and the live issue/source replaced that unavailable graph read. The KB synthesis describes admission integrity, not general process cancellation.

Created 2026-10-07 01:15:42Z; no stale/exemption labels. This is inside the Engine’s 90-day stale window; no Brain-local stale workflow is present. Same-day merged #910 was inspected rather than treated as a duplicate resolution. ADR successor-risk: aligned with the existing lifecycle/admission ownership; no config, credential audience, wire verb, or decision-record change.

Next proof: extend production-composer tests to require no spawn, lease activation, or later wake arming after Stop, with queued, non-Desktop, fresh later Start, other-seat, and existing-process cleanup controls. Source-only fixtures; no live harness experiment.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf

- 2026-10-08T01:47:58Z @neo-gpt-emmy cross-referenced by PR #926
- 2026-10-08T01:58:36Z @neo-gpt-emmy referenced in commit `b888c1a` - "test(fleet): update composed-start lifecycle doubles (#911)"
- 2026-10-08T09:42:06Z @neo-gpt-emmy referenced in commit `e4282a2` - "fix(fleet): retain canceled wake cleanup after client close (#911)"
- 2026-10-08T09:56:05Z @neo-gpt-emmy referenced in commit `f7417e9` - "fix(fleet): preserve Start cancellation across memory import (#911)"
- 2026-10-08T10:38:28Z @tobiu closed this issue
- 2026-10-08T10:38:28Z @tobiu referenced in commit `6de77a3` - "feat(fleet): cancel pending managed Starts (#911) (#926)

* feat(fleet): cancel pending managed Starts (#911)

* test(fleet): update composed-start lifecycle doubles (#911)"

