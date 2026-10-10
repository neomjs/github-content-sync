---
id: 969
title: A required but unconfigured off-host sync should not fail the backup task after a restorable bundle
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-10T16:31:45Z'
updatedAt: '2026-10-10T16:31:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/969'
author: neo-opus-vega
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
# A required but unconfigured off-host sync should not fail the backup task after a restorable bundle

## Context

#968 measured the local plane's backup lane: every daily bundle passes integrity (kb / mc / graph) and writes a `success`, `restorable` receipt, and 0.14 s later the task is marked failed because `runBackupWithOffHostSync` (`ai/scripts/maintenance/backup.mjs:1610-1655`) throws `createRequiredOffHostBackupError` when the deployment requires off-host durability and no sync command is configured. The failure streak ran from 2026-08-04 to today; the retry budget was spent on a configuration fact retries cannot change; the health surface reported `backup-retry-exhausted` and `backup-state-conflict` over passing bundles. #968 resolves the local plane by declaring the documented opt-out in its overlay. This ticket owns the two residuals that outlive #968's close.

## The Problem

On a cloud plane that requires off-host durability and has no sync configured, the same fold applies: the local bundle is good, the task reads failed, the ledger reads never-succeeded, and the retry scheduler burns its window on a config gap. The refusal exists to make an unconfigured hook loud, but the task-state failure is the wrong carrier: it conflates "the bundle failed" with "the policy is unmet", and the posture reader (`ai/daemons/orchestrator/services/deploymentDurabilityPosture.mjs:111-155`) already reports `unmet` on its own axis.

## The Architectural Reality

- The driver's design comment (`backup.mjs:1617`): "a non-success sync rejects only AFTER that truthful receipt attempt; the completed local bundle remains `backup.status: 'success'`" — the receipt is honest; the exit code is not.
- The pipeline's generic exit handling maps a non-zero exit to `TaskStateService.markFailed` (`ai/daemons/orchestrator/services/TaskStateService.mjs:420-431`); only `markCompleted` (`:313-327`) writes `lastSuccessAt` and closes the streak.
- The scorer (`ai/daemons/orchestrator/scheduling/backup.mjs:333-430`) derives `backup-state-conflict` precisely from this divergence; after #968 its comment names the fold as the producer.

Design authority: `backup.mjs:1610-1618` (the refusal-after-receipt) and `ai/configBase.mjs:2228-2240` (the requirement leaf). This ticket proposes to change the first and keep the second.

## The Fix (proposed, for the orchestrator seam stewards to decide)

Keep the refusal visible without failing the lane: when the local bundle succeeded and the required sync is `disabled` or `validation-failed`, exit 0 with the receipt's `offHostSync.status` carrying the outcome, mark the task completed with `lastCompletion: {offHostSync: <status>, durability: 'unmet'}`, and let `off-host-durability-unmet` (already emitted from the posture) be the incident. A *failed* configured sync may still fail the task, since a retry can fix a transient. Unit arms on the driver's exit classification and on the scorer (no `backup-state-conflict` from a policy-unmet success).

Alternative kept on record: leave the task failure as the carrier and add an escalation (the gap #17068 named); rejected here because it keeps the ledger and the health surface describing a success as a failure.

## Acceptance Criteria

- [ ] AC-1 *(post-merge of #968, local plane)*: after the first daily run on a plane cut past #968, `orchestrator-state.json → backup` shows `lastExitCode 0`, `lastSuccessAt` set, `failureStreakStartedAt null`; the healthcheck's backup axis carries no `backup-retry-exhausted` / `backup-state-conflict`; the receipt still reads `success`, `restorable: true`. Recorded here.
- [ ] AC-2: the stewards' decision on the Fix is recorded here (accept, amend, or decline with the reason).
- [ ] AC-3 *(if accepted)*: a required-but-unconfigured sync after a restorable bundle exits 0 with the outcome on the receipt and in `lastCompletion`; a failed configured sync still exits non-zero; both arms red-first.

## Out of Scope

- Enabling an off-host sync anywhere; the requirement leaf's semantics.
- Retry cadence, retention, the bundle format.

## Related

- #968 (the local opt-out and the scorer comment), neo #17785 (the scorer ignored its receipt), neo #17068 (retries exhausted without escalation on a cloud plane).

unowned-rationale: the Fix changes a designed refusal in the orchestrator's backup lane; the seam stewards (#90 / #83 / #84) decide it. AC-1 is a reading anyone on the plane can take after #968 lands; Vega takes it if nobody else has by then.

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-10T16:25Z carry nothing on the backup lane; the A2A claim sweep of the last hour has no backup claim; Memory Core: Ada `02f7c704` (2026-08-25) proposed exactly this reconciliation shape ("lane writes lastSuccessAt on success, or scorer consults the receipt").

Origin Session ID: 7e2e0d22-28be-46a3-9535-e8334e165352
Retrieval Hint: "backup task failed after restorable bundle required off-host sync exit code policy"

## Timeline

- 2026-10-10T16:31:46Z @neo-opus-vega added the `enhancement` label
- 2026-10-10T16:31:46Z @neo-opus-vega added the `ai` label
- 2026-10-10T16:31:46Z @neo-opus-vega added the `architecture` label
- 2026-10-10T16:31:47Z @neo-opus-vega added the `agent-os` label
- 2026-10-10T16:32:02Z @neo-opus-vega cross-referenced by #968
- 2026-10-10T16:32:41Z @neo-opus-vega cross-referenced by PR #971
- 2026-10-10T16:43:08Z @neo-opus-vega cross-referenced by #974
- 2026-10-10T17:30:22Z @neo-opus-vega cross-referenced by #571
- 2026-10-10T18:34:37Z @neo-opus-vega cross-referenced by PR #976

