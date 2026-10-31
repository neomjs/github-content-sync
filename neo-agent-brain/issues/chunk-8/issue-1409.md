---
id: 1409
title: The fleet deployment projection drops each service's start time
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-31T21:20:33Z'
updatedAt: '2026-10-31T21:51:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1409'
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
closedAt: '2026-10-31T21:51:40Z'
---
# The fleet deployment projection drops each service's start time

## Context

neomjs/neo-agent-institution#844 asks the FM cockpit's spine to name a plane outage that falls between two cockpit reads, once it has recovered. Its PR (neomjs/neo-agent-institution#854) detects a restart by comparing each service's start time across two `fleetDeploymentState` reads.

A review probe sent snapshot records, shaped as the plane's bridge writes them, through `projectDeploymentStateForFleet` (the Institution's pinned Brain, `9668697b`) and then into that detector. The probe used a plane cut on #844's own timeline: mc/kb/fleet-server recreated 21:59:03–21:59:14Z, the orchestrator at 21:59:37Z.

- Each projected service keeps `serviceKey, observedAt, status, memoryPressure, restartChurn, classification, diagnosis`. No start time survives.
- Read once the stats ring had refilled, the cut produced no restart at all.

The plane's own snapshot does record it. `get_deployment_state_snapshot` → `services[].inspect.currentRun.startedAt` is present on all five services of the live plane (2026-10-31).

## The Problem

The cockpit reads only `fleetDeploymentState`, and its projection is "the only shape `fleetDeploymentState` ever puts on the fleet wire". No field in that shape changes when a service restarts and stays changed until the next read. The cockpit reads deployment state every 120 s (`LivenessCadence.DEFAULT_INTERVALS.deploymentState`), so a restart between two of those reads is invisible to it.

The two nearest proxies misread:
- `classification.sampleCount` is the length of the orchestrator's stats ring (`statsSampleWindow` 2). The ring refills within two 30 s bridge writes, so a 120 s read sees the drop about a quarter of the time. After an orchestrator restart it drops for every service, including ones that did not restart.
- `restartChurn.detecting` reports whether the churn detector can work ("can this service produce a churn verdict at all right now?"), not whether a service restarted.

## The Architectural Reality

- `ai/services/fleet/projectDeploymentStateForFleet.mjs` `projectDeploymentServiceForFleet(row)` is the bounded per-service projection; each nested block keeps named fields only. It already lifts nested leaves under their own names: `diagnosis.recoveryClass` and `diagnosis.confidence` come from the decision's inner `diagnosis` block.
- In plane mode the FM's fleet process reads the plane's snapshot through `get_deployment_state_snapshot` (`planeDeploymentStateReader.mjs`) and projects it locally. The field therefore reaches the cockpit through the Brain that the FM runs its fleet process from, not through a plane cut.
- Writer: `DeploymentStateBridgeService` writes `inspect.currentRun.startedAt`, the current container run's start (ISO, nanosecond precision).
- Design authority: n/a — this adds one leaf and calls no existing behavior wrong. #314 binds the shape. Its Fix says the projection carries "Never `resolvedConfig`, `inspect`, `logs`, `proofs`, `stats`, `heapObservation`, or any absolute path / env value / container id beyond `serviceKey`", and "Field names come from the snapshot's own vocabulary; nothing is renamed". A start time is none of the redacted kinds, and lifting the leaf keeps `inspect` itself off the wire.
- Structure map (`npm run ai:structure-map -- --files --loc`, 2026-10-31): owning module `ai/services/fleet/projectDeploymentStateForFleet.mjs`; no new file.

## The Fix

In `projectDeploymentServiceForFleet`, add a service-level `startedAt`, read from `inspect.currentRun.startedAt` and `null` when the record lacks it. `inspect` stays unprojected. Extend the module summary by one line, naming the leaf and its reader (the cockpit's restart read). In `test/playwright/unit/ai/services/fleet/projectDeploymentStateForFleet.spec.mjs`, the leaking fixture's `inspect` gains a `currentRun.startedAt` beside its sensitive fields. The spec asserts that only that leaf survives, as `startedAt`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `fleetDeploymentState` → `services[].startedAt` (new leaf on the existing per-service projection) | writer `summarizeInspect` in `DeploymentStateBridgeService.mjs` (`inspect.currentRun.startedAt`); shape rule #314 | the service's current container start time, as the snapshot writes it | `null` when `inspect`, `currentRun` or `startedAt` is absent or not a string | `projectDeploymentStateForFleet.mjs` summary | projection spec: carry arm, absent arm, redaction arm |

## Decision Record impact

`none` (no ADR governs the projection). It refines #314's closed field list by one leaf, within that ticket's redaction rule.

## Acceptance Criteria

- [ ] AC-1 `projectDeploymentServiceForFleet` carries `startedAt` from `inspect.currentRun.startedAt`, and `null` when any link is absent (spec).
- [ ] AC-2 Nothing else from `inspect` survives: the leaking fixture's other `inspect` content (env, mounts, paths, container id) projects no leaf (spec).
- [ ] AC-3 The module summary names `startedAt` and its reader.

## Out of Scope

- The Institution consumer and its Brain pin bump: neomjs/neo-agent-institution#844, PR #854.
- Any other `inspect` field, and a restart verdict on the Brain side. The consumer compares two reads.

## Avoided Traps

- Treating `classification.sampleCount` or `restartChurn.detecting` as a restart signal (The Problem).
- Projecting an `inspect` block, even a bounded one: #314 keeps `inspect` off the wire, and a lifted leaf meets the need.

## Related

neomjs/neo-agent-institution#844 (blocked by this ticket) · neomjs/neo-agent-institution#854 · #314 (the projection and its field rule) · #323 (an earlier projection/writer mismatch).

Live latest-open + closed sweep: checked the latest 20 open issues plus closed/all searches ("deployment projection startedAt", "fleetDeploymentState restart") at 2026-10-31T21:18Z. Read the Fix, Out of Scope and Avoided Traps of #314; it excludes the `inspect` block and raw container detail, not a start time. A2A sweep (last 30, all read states, through 21:16Z): no claim on this scope. Memory Core sweep (problem nouns) and own-assignment sweep: no equivalent.

Ownership: self-assigned. It blocks a peer's open PR, and the gap is my own #844's premise (its route 2 assumed the cockpit could read restart times).

Retrieval Hint: "fleet deployment projection service start time restart detection cockpit spine"

Origin Session ID: 94dc6bfe-e91a-4ab9-9c0e-8cf1e2a2cf1e


## Timeline

- 2026-10-31T21:20:33Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-31T21:20:35Z @neo-opus-ada added the `enhancement` label
- 2026-10-31T21:20:35Z @neo-opus-ada added the `ai` label
- 2026-10-31T21:20:35Z @neo-opus-ada added the `agent-os` label
- 2026-10-31T21:21:46Z @neo-opus-ada cross-referenced by #844
- 2026-10-31T21:24:51Z @neo-opus-ada cross-referenced by PR #854
- 2026-10-31T21:30:40Z @neo-opus-ada cross-referenced by PR #1413
- 2026-10-31T21:51:40Z @tobiu referenced in commit `546e6ab` - "feat(fleet): the deployment projection carries each service's start time (#1409) (#1413)

The cockpit reads fleetDeploymentState only, and its per-service projection kept no field that changes on a restart and stays changed until the next read. A restart between two of its 120 s reads was invisible to it (Institution #844).

projectDeploymentServiceForFleet now lifts inspect.currentRun.startedAt, the container run's start, as a service-level startedAt: null when any link is absent or not a string. inspect itself stays off the wire, per #314's field rule. The spec's leaking fixture gains a currentRun beside its sensitive inspect fields, and the exact-shape arms show that only startedAt survives."
- 2026-10-31T21:51:41Z @tobiu closed this issue

