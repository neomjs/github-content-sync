---
id: 1667
title: Seat Starts wait forever on a runtime lease a live holder never frees
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-13T03:28:33Z'
updatedAt: '2026-11-13T05:07:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1667'
author: neo-opus-ada
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
closedAt: '2026-11-13T05:07:33Z'
---
# Seat Starts wait forever on a runtime lease a live holder never frees

## Context

Since 2026-11-13T02:51:27Z no seat can start from the installed FM (Institution e717bbf + Brain 857fe415). Grace's Start at ~02:56Z fetched her repos (FETCH_HEAD 02:56:55Z) and ran `seat:start` (AGENTS.md 02:56:56Z), then never launched; the cockpit showed `fleet: request transport failed`. A retry at 03:18:09Z showed `start timed out after 30000ms`, and the Fleet spawned no child in the next 7 minutes. Seats started before 02:51Z (02:31Z, 02:47Z) keep running.

Observed: `~/Library/Application Support/neo-harness/brain/runtime/.runtime-references.lock` holds `pid 99017, owner fleet-runtime-references, startedAt 02:51:27.999Z, lastPulse 02:51:28.005Z`. pid 99017 is the FM main: alive, 0.0–0.1% CPU, no file open under `runtime/` except its host pin. main.log has no `HARNESS_RUNTIME_*` line since 01:14:55Z.

## The Problem

Two defects compose into a fleet-wide freeze:

1. **A Start waits without a bound.** `startAgentProvisioned` awaits `acquireRuntimeReferences` (= `waitForRuntimeGenerationReferences`), a 25 ms retry loop that ends only when the Start is aborted. `fileLease` liveness is `process.kill(pid, 0)`; `lastPulse` is never read. A live holder that stops releasing therefore blocks every Start, for every harness type, until its process exits. The cockpit reports a transport failure or its own 30 s timeout, never the holder.
2. **The holder stopped releasing.** Institution's retirement pass hung inside its `try` while holding this lease and the installer's package lease (neomjs/neo-agent-institution#946, fixed by neomjs/neo-agent-institution#947).

Defect 1 is what turns one stuck holder into "no seat starts until the FM quits".

## The Architectural Reality

- `ai/services/fleet/startAgentProvisioned.mjs`: `runtimeReferenceGuard = installedRuntime ? await acquireRuntimeReferences({directory, signal: startSignal}) : null`. This runs on every packaged Start, between `installDependencies` and `admission.reserve`.
- `ai/services/fleet/runtimeGenerationLease.mjs#waitForRuntimeGenerationReferences`: loops while `FileLeaseHeldError` names a holder.
- `ai/daemons/shared/fileLease.mjs#defaultIsHeldFresh`: pid probe only, although the module already carries `pulse()` and `lastPulse`.
- Other waiters: `FleetLifecycleService#recoverRuntimeCustody` (FM boot), Institution `acquireHostRuntimeGeneration` and `harness/install.mjs`.
- A Start's runtime refusal reaches the cockpit only in the reply to its request, mapped by `FleetControlBridge#startOutcome` (unnamed errors become the generic `operation-failed`). `launchRefusal` carries static refusals.

Design authority: the waiter's own contract says it "lets an existing cancelable Start wait for known contention without creating a retry obligation". That assumes the contention ends; this ticket bounds it for the Start.

## The Fix

- `waitForRuntimeGenerationReferences` takes an opt-in `deadlineMs` (default `null`, unbounded). At the deadline it throws `RuntimeReferencesHeldError` (`code: 'RUNTIME_REFERENCES_HELD'`), naming the holder's `owner`, `pid`, `startedAt` and `lastPulse`, never its token.
- `FleetControlBridge#startOutcome` answers it as a domain rejection carrying its message and code; unnamed errors stay sanitized.
- The Start passes `RUNTIME_REFERENCE_START_WAIT_MS` = 20 s as a bound on the lease wait itself, longer than a healthy pass (a payload verify takes about 6 s, a census at most 10 s). The wait begins after repository preparation and dependency installation, so the cockpit's 30 s round trip shows the refusal when those finished within about 10 s.

## Acceptance Criteria

- [ ] A Start against a lease held by a live holder that never releases refuses within its bound and names the holder: hangs on base, refuses at head.
- [ ] Contention that ends inside the bound still enters.
- [ ] Every other waiter keeps its unbounded wait.

## Post-Merge Validation

- [ ] FM build + install, then a Start during an induced long hold shows the named refusal in the cockpit.

## Avoided Traps

- **Reclaiming a lease whose holder lives but whose pulse is stale** (this ticket's first draft). Dropped: Institution #947 bounds the holder that hung. A stale-pulse rule would also make a slow but alive holder, such as the installer, reclaimable mid-operation, which breaks the exclusion this lease exists for.

## Out of Scope

- Why the retirement pass hung, and its deadline: neomjs/neo-agent-institution#946 / #947.
- Bounds for the boot and installer waiters.
- Lease consolidation across the daemons: #72.

## Immediate unblock (operator; not this PR)

Recorded once on neomjs/neo-agent-institution#946 under "Unblock on the live FM". Done 03:38Z: Grace started again.

## Related

#72 · #1531 · neomjs/neo-agent-institution#946 · defect-note `9fbaad41905f3d08`

Live latest-open + closed sweep: latest 20 open in Brain and Institution, plus an all-state search (runtime-references, runtimeGenerationLease, retireRuntimeGenerations, runtime retirement, reference lease) at 2026-11-13T03:35Z; nearest is #72 (consolidation scope), no equivalent. A2A: no claim on this scope in the latest 25; Euclid's 03:18Z crisis intake is read-only. Memory: no prior hit. Backlog: 158 open (red); no owning epic, since #1531 covers vendor harness generations, not the FM's own runtime generations.

Origin Session ID: 2893c284-25f8-411c-b5dc-f37eafd573fe
Retrieval Hint: "seat Start hangs on runtime-references lease held by the FM main"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-13T03:28:35Z @neo-opus-ada added the `bug` label
- 2026-11-13T03:28:35Z @neo-opus-ada added the `ai` label
- 2026-11-13T03:28:35Z @neo-opus-ada added the `agent-os` label
- 2026-11-13T03:28:41Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-13T03:29:04Z @neo-opus-ada cross-referenced by #946
- 2026-11-13T03:55:50Z @neo-opus-ada cross-referenced by PR #947
- 2026-11-13T04:07:27Z @neo-opus-ada cross-referenced by PR #1668
- 2026-11-13T04:26:23Z @neo-opus-ada referenced in commit `b3732a0` - "fix(fleet): a held-lease Start refusal reaches the wire (#1667)

startOutcome rethrew RuntimeReferencesHeldError, so the dispatcher
answered the generic operation-failed and the cockpit never saw the
holder. It now matches the error by name and code, as it does the
workspace-preparation failure, and answers its own message (holder
named, no lease token) with its code. Unnamed errors stay sanitized.

The wait-bound docblock now says the 20 s begins after repository
preparation and dependency installation, so the cockpit's 30 s round
trip shows the refusal only when those finished within about 10 s."
- 2026-11-13T04:54:09Z @neo-opus-ada cross-referenced by #1531
- 2026-11-13T05:07:33Z @tobiu referenced in commit `bade95f` - "feat(fleet): a Start refuses a runtime lease held too long (#1667) (#1668)

* feat(fleet): a Start refuses a runtime lease held too long (#1667)

A seat Start waited for the runtime-reference lease in a 25 ms loop
with no deadline, and fileLease reads a live pid as held forever. On
2026-11-13 a retirement pass that never settled kept every seat from
starting until the FM process ended; the cockpit saw only "transport
failed" or its own 30 s timeout.

waitForRuntimeGenerationReferences takes an opt-in deadlineMs and then
throws RUNTIME_REFERENCES_HELD naming the holder's owner, pid, start
and last progress (never its token). The Start passes 20 s, inside the
cockpit's 30 s round trip. Other callers keep waiting until release.

* fix(fleet): a held-lease Start refusal reaches the wire (#1667)

startOutcome rethrew RuntimeReferencesHeldError, so the dispatcher
answered the generic operation-failed and the cockpit never saw the
holder. It now matches the error by name and code, as it does the
workspace-preparation failure, and answers its own message (holder
named, no lease token) with its code. Unnamed errors stay sanitized.

The wait-bound docblock now says the 20 s begins after repository
preparation and dependency installation, so the cockpit's 30 s round
trip shows the refusal only when those finished within about 10 s."
- 2026-11-13T05:07:35Z @tobiu closed this issue

