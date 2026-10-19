---
id: 742
title: A rejected Start's reason is not overwritten by cleanup on roster re-poll
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-19T12:51:24Z'
updatedAt: '2026-10-19T12:51:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/742'
author: neo-gemini-maryam
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
---
# A rejected Start's reason is not overwritten by cleanup on roster re-poll

## Context

Observed on the installed Fleet Manager (`Brain c25e1015` + `Institution e2e10e0d`, 2026-10-19) while starting `neo-gpt-emmy`:
1. The seat starts in `{state: 'failed', cleanupUnresolved: true}`, showing `⚠ cleanup: the original runtime seat could not be re-adopted — Start re-checks it`.
2. Clicking **Start** fails (`status: 'rejected'`) and renders `⚠ rejected: refusing to spawn agent 'neo-gpt-emmy' while Codex Desktop...` on the card's control-status line.
3. ~1 second later, the post-settle roster refresh (`refreshRosterOnSettle` → `loadRoster()`) overwrites that rejection with `⚠ cleanup: the original runtime seat could not be re-adopted — Start re-checks it`.

## The Problem

`FleetLifecycleIntentAdapter.rosterMayRead` (`apps/agentos/util/FleetLifecycleIntentAdapter.mjs:91–100`) triggers a post-settle roster read on an accepted-then-rejected Start so that seat fields written during the Start attempt (`gitIdentity`, `seatModel`, `repoOutcomes`) refresh on the card, relying on the invariant documented in its JSDoc:
- `"A re-poll keeps the card's own reason: the roster rows carry none."`

PR `#728` (for `#725`) changed `SourceHealth.mapFleetSessionHealth` (`apps/agentos/util/SourceHealth.mjs:277–286`) and `RosterRow.mapRosterRow` (`apps/agentos/util/RosterRow.mjs:91`) so that any polled roster row with `{state: 'failed', cleanupUnresolved: true}` carries `controlReason: {action: 'stop', kind: 'cleanup', reason}`. When a Start attempt on such a seat is rejected (or while a lifecycle action is in flight or timed out), the seat's backend row on `/fleet/agents` still reports `{state: 'failed', cleanupUnresolved: true}`. As a result, `LivenessController.reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:665–673`) calls `record.set(next)` with `next.controlReason = {action: 'stop', kind: 'cleanup', reason}`, clobbering the C2 adapter's active `record.controlReason` (`rejected` / `timeout` / `unauthorized` / `superseded`, or `null` while `record.pendingAction` is in flight) ~1s after the button press.

## The Architectural Reality

- `Design authority: "Whether a settled round-trip can have changed what the roster reads: a lifecycle change, or a refusal the Fleet answered, which may record its cause on the seat. Never a timeout, whose outcome is unknown, nor a rejection that never left this cockpit. A re-poll keeps the card's own reason: the roster rows carry none."` (`apps/agentos/util/FleetLifecycleIntentAdapter.mjs:91–93`) and `"Reconcile an authoritative roster snapshot onto the store: known ids update in place (record.set — one card re-render, producer-foreign fields survive)"` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:637–638`).
- `apps/agentos/view/fleet/cockpit/LivenessController.mjs:665–673` currently clears `record.controlReason` when `!next.controlReason && record.controlReason?.kind === 'cleanup'`, but does not guard an existing C2 `record.controlReason` (`record.controlReason.kind !== 'cleanup'`) or an in-flight `record.pendingAction` against being overwritten by a snapshot row's passive `next.controlReason?.kind === 'cleanup'`.

## The Fix

In `LivenessController#reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:665–673`), when `next.controlReason?.kind === 'cleanup'` and the existing record either has an in-flight lifecycle action (`record.pendingAction`) or holds a non-cleanup `record.controlReason` (`record.controlReason && record.controlReason.kind !== 'cleanup'`), omit `controlReason` from `next` (`delete next.controlReason`) so `record.set(next)` updates all roster-owned fields (`gitIdentity`, `seatModel`, `repoOutcomes`, `cleanupUnresolved`, etc.) while preserving the C2 control outcome until the next lifecycle attempt or until cleanup resolves.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|\n| `LivenessController#reconcileRoster` (`apps/agentos/view/fleet/cockpit/LivenessController.mjs:653–685`) | `FleetLifecycleIntentAdapter.mjs:91–93` · `LivenessController.mjs:637–638` · `CARD-CONTRACT.md:21` | Snapshot row's `controlReason: {kind: 'cleanup'}` applies on cold admission or when `record` has no active `pendingAction` and no non-cleanup `controlReason`; never overwrites an active C2 `controlReason` (`rejected` / `timeout` / `unauthorized` / `superseded`) or an in-flight `pendingAction` | Clears `record.controlReason` when `!next.controlReason && record.controlReason?.kind === 'cleanup'` | JSDoc on `LivenessController#reconcileRoster` and `FleetLifecycleIntentAdapter.rosterMayRead` | Unit test in `rosterStore.spec.mjs` |

Decision Record impact: none (restores the C2 `controlReason` survival invariant in `LivenessController#reconcileRoster`).

## Acceptance Criteria

- [ ] AC-1: In `LivenessController#reconcileRoster`, a snapshot row carrying `controlReason: {action: 'stop', kind: 'cleanup', reason}` does not overwrite an existing record's non-cleanup `controlReason` (`rejected`, `timeout`, `unauthorized`, `superseded`) or re-introduce a `cleanup` reason while `record.pendingAction` is non-null, while still updating all other roster fields on the record.
- [ ] AC-2: Cold admission (`joiners`), an idle record with `controlReason === null && !pendingAction`, and cleanup resolution (`!next.controlReason && record.controlReason?.kind === 'cleanup'` → `null`) continue to set/update/clear the `cleanup` `controlReason` as required by `#725`.
- [ ] AC-3: Unit coverage in `test/playwright/unit/apps/agentos/view/fleet/cockpit/rosterStore.spec.mjs` verifies the exact Start-rejected → `refreshRosterOnSettle` / `reconcileRoster` sequence with `{state: 'failed', cleanupUnresolved: true}`, confirming `record.controlReason` (`kind: 'rejected'`) survives the reconcile and clears when a subsequent attempt starts and settles cleanly.

## Out of Scope

- Backend `FleetLifecycleService` helper-cleanup probing (`neomjs/neo-agent-brain#1183` / `#1184`).
- Modifying `SourceHealth.mapFleetSessionHealth` or `RosterRow.mapRosterRow` for initial cold admission.

## Avoided Traps

- Removing `controlReason` from `RosterRow.mapRosterRow` entirely: cold-loaded cards for seats with `cleanupUnresolved: true` need the initial `cleanup` `controlReason` from the roster snapshot (`#725`). The collision is strictly on in-place `reconcileRoster` when the C2 adapter has written an active `pendingAction` or non-cleanup `controlReason`.

## Related

- `#725` / PR `#728` (introduced snapshot-borne `controlReason: {action: 'stop', kind: 'cleanup', reason}`).
- `neomjs/neo-agent-brain#1183` / PR `#1184` (where the Start rejection on `neo-gpt-emmy` was observed).

Live latest-open sweep: checked latest 20 open Institution issues at 2026-10-19T12:48Z (newest `#739`); no equivalent found. A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-19T12:48Z; no competing claim on `LivenessController#reconcileRoster` or `controlReason` overwrite. Memory Core rationale sweep: `query_raw_memories` on `Start failure error reason overwritten after 1 second cleanup controlReason reconcileRoster` returned only our own root-cause trace (`0e69eebf`). Own-assignment sweep: `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-institution`. Structure map: N/A (Institution).

Origin Session ID: 623823b7-3c27-4f59-a8e9-b04710554494

Retrieval Hint: "LivenessController reconcileRoster cleanup controlReason overwrites rejected Start reason refreshRosterOnSettle"


## Timeline

- 2026-10-19T12:51:25Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-19T12:51:26Z @neo-gemini-maryam added the `bug` label
- 2026-10-19T12:51:26Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-19T12:51:26Z @neo-gemini-maryam added the `ai` label
- 2026-10-19T13:06:32Z @neo-gemini-maryam cross-referenced by PR #744

