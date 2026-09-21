---
id: 323
title: fleetDeploymentState reads a service status object the snapshot never writes
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-05T00:48:44Z'
updatedAt: '2026-09-05T11:58:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/323'
author: neo-fable-clio
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
closedAt: '2026-09-05T11:58:39Z'
---
# fleetDeploymentState reads a service status object the snapshot never writes

## Context

The `fleetDeploymentState` wire verb (#314, shipped in PR #315) projects the orchestrator's deployment-state snapshot for the Fleet Manager's System view (neomjs/neo-agent-institution#21). Building that view's consumer against the LIVE plane (snapshot `generatedAt` 1788568958677, read 2026-09-05T00:42:38Z via `get_deployment_state_snapshot`) shows the projection was written against a guessed snapshot shape, not the writer's: every one of the five services projects `status: null`, so every plane card would read *unobserved* on a plane whose services are all `available` — and the backup lane's phase projects `null` on a plane whose lane is `exhausted`.

## The Problem

`projectDeploymentServiceForFleet` (`ai/services/fleet/projectDeploymentStateForFleet.mjs:39-73`) assumes:

- `row.status` is a block `{status, disposition}` — it reads `nestedOf(row, 'status')`. The writer emits a **string**: `DeploymentStateBridgeService.mjs:965` sets `status: foldMemoryPressureIntoStatus({status: errors.length > 0 ? 'degraded' : 'available', disposition})`, and the fold (`memoryPressureDisposition.mjs:252`) returns `'available' | 'degraded'`. The disposition lives on `memoryPressure.disposition` (`below | at-cap | unknown`), which the projection already reads correctly. Live rows: `"status": "available"` ×5 → projected `null` ×5.
- `classification.serviceClassDeclared` is the class — it is a **boolean** flag; the class word is `classification.serviceClass` (`store`, `transient`, … — live: chroma `store`, the four Node services `transient`).
- `diagnosis` carries `recoveryClass` / `confidence` at its root and `details.actionClass` / `details.classificationReason` — the record is a `container-health-diagnosis-decision` (`ContainerHealthDiagnosisService.createDecision`): `{status, actionClass, diagnosis: {diagnosisId, recoveryClass, confidence, …} | null, facts}`. There is no `details` block; `recoveryClass` and `confidence` sit under `diagnosis.diagnosis` and are `null`-absent on a healthy service (live: `diagnosis.status: "healthy"`, `actionClass: null`, `diagnosis: null`).

**Scope addition (found during the fix, same defect class):** `projectDeploymentMaintenanceForFleet` reads the backup lane's `phase`, `lastSuccessAt` and `lastSuccessAgeMs` from `maintenance.health` — the writer puts them on `maintenance.retry` (`describeBackupRetryState`, `scheduling/backup.mjs:220`: `phase` ∈ `healthy | unanchored | retrying | exhausted`); `maintenance.health` is the verdict `{status, observationStatus, reasonCodes[], staleAfterMs}`; and the receipt `maintenance.lastBackup` carries `backup.status` (`success`) and `offHostSync.status`, not a root `kind`/`status` (those exist only on the unreadable/unreachable shape). Live: `retry.phase: "exhausted"`, `retry.lastSuccessAt: null`, `health.status: "degraded"`, `health.reasonCodes: [off-host-durability-unmet, backup-retry-exhausted, backup-never-succeeded]`, `lastBackup.backup.status: "success"`, `lastBackup.offHostSync.status: "disabled"` → the old projection gave `phase: null`, `lastBackup.kind: null`, `lastBackup.status: null`.

Observation vs inference: the null projections are observed on the live snapshot; the field reads are inferred from the source lines cited; the writer shapes are read from the writers, not from the spec fixtures (which were hand-written to the guessed shapes and passed).

## The Architectural Reality

- Writer authority: `ai/daemons/orchestrator/services/DeploymentStateBridgeService.mjs` (`recordType: 'deployment-service-state'`, schemaVersion 1; `collectMaintenanceSnapshot` → `retry` / `health` / `lastBackup`), `ai/daemons/orchestrator/services/ContainerHealthDiagnosisService.mjs` (`createDecision`) and `ai/daemons/orchestrator/scheduling/backup.mjs` (`describeBackupRetryState`, `describeBackupMaintenanceHealth`).
- Projection: `ai/services/fleet/projectDeploymentStateForFleet.mjs` — bounded, redacted, observe-only (#314's R3 seam); `memoryPressure`, `restartChurn` and `starvation` are projected correctly and stay.
- Consumer: the Institution twin (`apps/agentos/util/DeploymentStateRead.mjs`, neomjs/neo-agent-institution#21) lands the rows as one atomic array and renders per-card state words from `status` + `memoryPressure.disposition` + `classification.serviceClass` + `diagnosis`; the lanes render `backup.phase` / `backup.health` / `backup.lastBackup` and `starvation`.

## The Fix

In `projectDeploymentServiceForFleet`:

- `status: typeof row.status === 'string' ? row.status : null` (the folded word; no disposition sub-field).
- `classification: {serviceClass, serviceClassDeclared, appliedMemoryThreshold, sampleCount}` (add the word, keep the flag).
- `diagnosis: decision && {status, actionClass, recoveryClass: decision.diagnosis?.recoveryClass ?? null, confidence: decision.diagnosis?.confidence ?? null}` — `classificationReason` and `details` go (they never existed on the wire).

In `projectDeploymentMaintenanceForFleet`:

- `backup: {phase, lastSuccessAt, lastSuccessAgeMs}` from `maintenance.retry`; `backup.health: {status, reasonCodes}` from `maintenance.health` (string codes only); `backup.lastBackup: {finishedAt, status, offHostSync}` with `status` from `backup.status` falling back to the unreadable shape's root `status`. `kind` goes (a receipt has none).

Spec: the service-row fixture becomes a VERBATIM live row (`chroma`, `status: "available"`, `serviceClass: "store"`, healthy decision) plus one degraded row whose decision carries `diagnosis: {recoveryClass: 'exhaustion', confidence: 0.95}` and `actionClass: 'restart'`; the maintenance fixture takes the writer's `retry` / `health` / receipt shape, with the leak markers extended to the receipt's own private fields (`durability` block, `stagingResidue`, `stderrTail`, `bundleName`, `targetPath`) — red-first against the current projection. JSDoc on both projections names the writers as the shape authority.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `fleetDeploymentState.services[].status` (wire, #314) | `DeploymentStateBridgeService.mjs:965` + `memoryPressureDisposition.mjs:252` | the folded string `available \| degraded` | a non-string → `null` (consumer renders *unobserved*, today's behaviour, never a throw) | JSDoc | projection spec, live-row fixture |
| `services[].classification.serviceClass` (new field) | the diagnosis classification block | the declared class word | absent → `null` | JSDoc | same spec |
| `services[].diagnosis` | `ContainerHealthDiagnosisService.createDecision` | `{status, actionClass, recoveryClass, confidence}` | healthy decision → `recoveryClass`/`confidence` `null`; no decision → `null` | JSDoc | same spec + a degraded fixture |
| `maintenance.backup` | `collectMaintenanceSnapshot` (`retry`, `health`, `lastBackup`) | `{phase, lastSuccessAt, lastSuccessAgeMs, health: {status, reasonCodes}, lastBackup: {finishedAt, status, offHostSync}}` | no `retry` yet → `phase` `null`; unreadable receipt → its root `status`, `offHostSync` `null` | JSDoc | same spec, the writer-shaped maintenance fixture + an unreadable-receipt arm |

Decision Record impact: none (a projection reads the writer it already claims to project).

## Acceptance Criteria

- [ ] `projectDeploymentStateForFleet.spec.mjs` carries a verbatim live service row and asserts `status === 'available'`, `classification.serviceClass === 'store'`, `diagnosis.status === 'healthy'`, `diagnosis.actionClass === null`, `diagnosis.recoveryClass === null` — red on the current projection, green after.
- [ ] A degraded fixture (`status: 'degraded'`, decision with `recoveryClass: 'exhaustion'`, `confidence: 0.95`, `actionClass: 'restart'`) projects all four.
- [ ] `fleetServer.spec.mjs`'s composed-path receipt and `createDeploymentStateReadSource.spec.mjs` stay green (`memoryPressure`, `restartChurn`, `starvation` untouched).
- [ ] The projections' JSDoc names the writer files as the shape authority.
- [ ] `maintenance.backup` projects the writer's `retry.phase` / `retry.lastSuccessAt` / `retry.lastSuccessAgeMs`, `health.{status, reasonCodes}` and the receipt's `finishedAt` / `backup.status` / `offHostSync.status`; a block without `retry` projects a `null` phase and keeps the verdict; an unreadable receipt reads its root `status`; the leak arm rejects the receipt's private fields.

## Out of Scope

The Institution's plane cards and lanes (neomjs/neo-agent-institution#21 renders the corrected shapes) · the snapshot schema itself · `memoryPressure` / `restartChurn` / `starvation` (correct today) · a service catalog / label on the wire.

## Related

#314 (the verb) · PR #315 (shipped the guessed shapes) · #322 (sibling reducer over the same snapshot) · #324 (fixture sibling, same PR) · neomjs/neo-agent-institution#21 (consumer) · neomjs/neo-agent-institution#113.

Live latest-open sweep: latest 20 open Brain issues read 2026-09-05T00:47:37Z, no equivalent (keyword sweep on the projection returns only #314, closed). A2A in-flight claim sweep: mailbox listing 2026-09-05T00:36Z, no claim on this surface. Memory Core sweep (`query_raw_memories`, the null-status symptom) returned no prior decision. Own-assignment sweep: my open Brain tickets (#322, #318, #37, #50, #51, #53) touch no projection shape.

Origin Session ID: 49133900-1f86-4134-a82b-30ff0709bcaf

Retrieval Hint: `query_raw_memories("fleetDeploymentState service status string folded projection null unobserved maintenance retry phase")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49133900-1f86-4134-a82b-30ff0709bcaf


## Timeline

- 2026-09-05T00:48:44Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-05T00:48:46Z @neo-fable-clio added the `bug` label
- 2026-09-05T00:48:46Z @neo-fable-clio added the `ai` label
- 2026-09-05T00:54:44Z @neo-fable-clio cross-referenced by #324
- 2026-09-05T00:57:49Z @neo-fable-clio cross-referenced by PR #325
- 2026-09-05T01:05:28Z @neo-fable-clio referenced in commit `1c30deb` - "fix(fleet): the maintenance projection reads the writer's retry, health and receipt blocks (#323)"
- 2026-09-05T11:35:06Z @neo-gpt-emmy cross-referenced by PR #114
- 2026-09-05T11:58:39Z @tobiu referenced in commit `dd2109f` - "Merge pull request #325 from neomjs/agent/323-service-status-projection

fix(fleet): fleetDeploymentState projects the writer's service row shape (#323)"
- 2026-09-05T11:58:39Z @tobiu closed this issue
- 2026-09-05T12:13:51Z @neo-fable-clio cross-referenced by #21

