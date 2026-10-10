---
id: 968
title: 'Backup exits 1 after a restorable bundle: off-host sync counts as required by the cloud default'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-10T16:27:05Z'
updatedAt: '2026-10-10T17:22:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/968'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-10-10T17:22:19Z'
---
# Backup exits 1 after a restorable bundle: off-host sync counts as required by the cloud default

## Context

Every Memory Core healthcheck on the local plane reads the backup axis as `degraded` with three codes: `off-host-durability-unmet`, `backup-retry-exhausted`, `backup-state-conflict`. Three readers took that at face value today (it blocked my #417 precondition); the operator corrected the same misreading on 2026-09-18 (Vega, Memory Core `a3838db1`) and Ada diagnosed the ledger-vs-receipt divergence on 2026-08-25 (`02f7c704`, under neo #17785). Measured 2026-10-10 on the plane serving Brain `93079328`:

| record | says |
|---|---|
| `~/.neo-ai/backups/last-backup-receipt.json` | `backup.status: success`, `integrity.restorable: true`, `emptySubsystems: []`, bundle `backup-2026-10-10T13-15-59.709Z`, finished 13:19:36.519Z, `offHostSync.status: "disabled"` |
| that bundle's `bundle-meta.json` | kb 120,911 / 120,911, mc 49,886 / 49,886, graph 578,954 / 578,954, all three integrity `pass`; 9.5 GB; one such bundle every day since 2026-09-10 |
| `orchestrator-state.json` → `backup` | `lastExitCode: 1`, `lastErrorAt: 13:19:36.568Z`, `lastSuccessAt: null`, `failureStreakStartedAt: 2026-08-04T08:22:24.707Z`, `interruptedAt: 2026-09-18T08:36:35.690Z` |

The receipt and the ledger describe the same run 0.14 s apart: the bundle succeeded and the task was marked failed.

## The Problem

`ai/configBase.mjs:1309` defaults `orchestrator.deploymentMode` to `cloud` ("the canonical Agent OS posture … unless a host process explicitly opts into `local`"), and the local plane sets no `NEO_AI_DEPLOYMENT_MODE`, which is correct: the Docker plane is the canonical posture. `orchestrator.cloudOnly.offHostBackupRequired` (`:2240`) is `null`, so `resolveCloudOnlyDefault(null, 'cloud')` (`ai/daemons/orchestrator/services/deploymentDurabilityPosture.mjs:75-78`) resolves **required**. The driver's `runBackupWithOffHostSync` (`ai/scripts/maintenance/backup.mjs:1631-1655`) then finds `offHostSync.status: 'disabled'` (no command configured) and, by design ("a non-success sync rejects only AFTER that truthful receipt attempt", `:1617`), throws `createRequiredOffHostBackupError` (`:146`) after writing the success receipt. The process exits 1; the pipeline's generic exit path calls `TaskStateService.markFailed` (`ai/daemons/orchestrator/services/TaskStateService.mjs:420-431`), which keeps the streak open; `lastSuccessAt` is written only by `markCompleted` (`:313-327`), which this lane has not reached since 2026-08-04.

Downstream, `describeBackupMaintenanceHealth` (`ai/daemons/orchestrator/scheduling/backup.mjs:333-430`) derives `backup-retry-exhausted` from the streak, `backup-state-conflict` from "success receipt beside `lastSuccessAt: null`", and `off-host-durability-unmet` from the posture. Its comment at `:407-416` attributes the conflict to "a lane that writes its receipt without going through the task-state path" and asks for the writer to be fixed. That attribution is wrong: the lane goes through the path and is marked failed on purpose, because the required-durability refusal is folded into the exit code. Retries cannot change a missing off-host command, so the retry budget is spent on a configuration fact, the ledger reports a two-month failure streak over daily restorable bundles, and every consumer of the health surface reads a false incident.

## The Architectural Reality

- The requirement is a tri-state leaf whose JSDoc (`ai/configBase.mjs:2228-2240`) already states the intended local answer: "local profile defaults not-required (the operator's own machine is not a durability boundary we can reason about) … an explicit `false` is a deliberate opt-out, which is what distinguishes it from an unconfigured hook nobody noticed."
- The local profile is `deploy/cloud/docker-compose.local-agent-os.yml` (orchestrator env block at `:110-119`). It declares no `NEO_ORCHESTRATOR_OFF_HOST_BACKUP_REQUIRED`, so the cloud default applies. Flipping `deploymentMode` to `local` is the wrong lever: that profile disables the container plane's maintenance ownership (`taskAuthority.mjs:20`, the wake daemon's refusal at `daemon.mjs:2277-2298`).
- The posture reader (`deploymentDurabilityPosture.mjs:111-155`) distinguishes `optedOut = offHostBackupRequired === false` from unconfigured, so an explicit `false` reports as a deliberate opt-out rather than `unmet`.

Design authority: `ai/configBase.mjs:2228-2240` (quoted above) for the opt-out; `ai/scripts/maintenance/backup.mjs:1610-1618` for the refusal-after-receipt design, which this ticket does not overturn.

## The Fix

1. **The local profile declares its documented default**: `NEO_ORCHESTRATOR_OFF_HOST_BACKUP_REQUIRED: "false"` in the overlay's orchestrator environment, with a two-line comment naming the leaf's rationale. Applied to the plane by the next cut; the next daily run then exits 0, `markCompleted` closes the streak, and the health axis drops `backup-retry-exhausted` and `backup-state-conflict`; `off-host-durability-unmet` reads as an opt-out.
2. **The observer comment names the real producer**: `scheduling/backup.mjs:407-416` points at the exit-code fold in `runBackupWithOffHostSync`, not at a receipt writer bypassing the ledger.
3. **Decision owed, not changed here**: whether a required-but-unsatisfied off-host sync should fail the *task* at all on cloud planes, where it burns the retry budget on a config fact and reports the local bundle as a failure. The alternative is `markCompleted` with the durability outcome in `lastCompletion` and the posture carrying the incident alone. Routed to the orchestrator seam stewards (#90 / #83 / #84, Euclid) as a follow-up; the refusal stays as designed until then.

## Acceptance Criteria

- [ ] AC-1: the local overlay sets `NEO_ORCHESTRATOR_OFF_HOST_BACKUP_REQUIRED` to `false` with the rationale, and the overlay's existing conformance specs stay green.
- [ ] AC-2: the `scheduling/backup.mjs` comment on `backup-state-conflict` names the exit-code fold as the producer and this ticket's opt-out as the local resolution.
- [ ] AC-3 *(post-merge, plane)*: after the first daily run on a plane cut past this change, `orchestrator-state.json → backup` shows `lastExitCode 0`, `lastSuccessAt` set, `failureStreakStartedAt null`; the healthcheck's backup axis carries no `backup-retry-exhausted` / `backup-state-conflict`; the receipt still says `success`, `restorable: true`.
- [ ] AC-4: the follow-up decision in Fix 3 is filed or declined by its stewards, linked here.

## Out of Scope

- Enabling an off-host sync for this laptop (an operator durability decision; the leaf is the place to flip it back to required once a command exists).
- Changing `deploymentMode`, the retry budget, or the scorer's code paths.
- Chroma, the bundle format, retention.

## Avoided Traps

- Treating the health reading as the incident: the bundles are restorable every day; the instrument's ledger was the only thing broken (operator correction of 2026-09-18, Memory Core `a3838db1`).
- Hand-editing the plane's env file: the overlay is the tracked home of the local profile; an env-file line would be a second, invisible copy.
- "Fix the writer" per the observer's comment: the writer is the designed refusal, not a bypass.

## Related

- neo #17785 (the scorer ignored its own success receipt, closed), neo #17068 (a cloud plane's retries exhausted without escalation, closed), #417 (its precondition reads this axis), neo #17647 AC-4 (the non-vacuity rule the scorer cites).
- Owning folders (structure map, exit 0): `ai/daemons/orchestrator/scheduling`, `ai/daemons/orchestrator/services`, `ai/scripts/maintenance`, `deploy/cloud`.

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-10T16:25Z, none on the backup axis (`gh search issues "backup-state-conflict"` returns only closed #17785 and unrelated tickets). A2A claim sweep (all read-states, last 60 min): no claim touches backups. Memory Core sweep: Ada `02f7c704` (2026-08-25), Vega `a3838db1` (2026-09-18) and `5d487650` (2026-08-13, #17068) name the symptom; none filed the overlay default. Own-assignment sweep: #417, #503, #65, #23, #28, #30, #64, #471, #493, #875 — none covers it.

Origin Session ID: 7e2e0d22-28be-46a3-9535-e8334e165352
Retrieval Hint: "backup exits 1 after success off-host required cloud default local overlay opt-out"

## Timeline

- 2026-10-10T16:27:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-10T16:27:06Z @neo-opus-vega added the `bug` label
- 2026-10-10T16:27:06Z @neo-opus-vega added the `ai` label
- 2026-10-10T16:27:07Z @neo-opus-vega added the `agent-os` label
- 2026-10-10T16:31:46Z @neo-opus-vega cross-referenced by #969
### @neo-opus-vega - 2026-10-10T16:32:01Z

Fix 3 and AC-3 / AC-4's post-merge readings now live in #969, which outlives this ticket's close: the stewards' decision on failing the task for a policy posture, and the local plane's ledger reading after the opt-out lands.

— Vega (Claude Fable 5.1, Claude Code) 🌿


- 2026-10-10T16:32:41Z @neo-opus-vega cross-referenced by PR #971
- 2026-10-10T16:41:16Z @neo-fable-clio cross-referenced by #973
- 2026-10-10T16:43:08Z @neo-opus-vega cross-referenced by #974
- 2026-10-10T17:22:19Z @tobiu closed this issue
- 2026-10-10T17:22:19Z @tobiu referenced in commit `8321ac2` - "fix(deploy): the local profile opts out of the required off-host backup (#968) (#971)

The Docker plane runs as deploymentMode cloud by design, and with offHostBackupRequired left null that resolves to required. The local plane configures no off-host sync, so backup.mjs exited 1 after every restorable bundle; the task ledger carried a failure streak since 2026-08-04 and the health surface reported backup-retry-exhausted and backup-state-conflict over daily passing bundles. The overlay now declares the documented local default, and the scorer's comment names the exit-code fold as the producer instead of a receipt writer bypassing the ledger. Three decayed ticket refs in the touched comments rewritten as prose."
- 2026-10-10T17:30:22Z @neo-opus-vega cross-referenced by #571

