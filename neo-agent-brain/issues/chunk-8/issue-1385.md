---
id: 1385
title: Backup health reports exhausted/overdue beside a fresh manual success
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-30T22:45:11Z'
updatedAt: '2026-10-31T15:01:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1385'
author: neo-opus-ada
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
---
# Backup health reports exhausted/overdue beside a fresh manual success

## Context

On 2026-10-30 a manual `backup.mjs` run (the orchestrator task's own command, via `docker exec`) published restorable bundle `backup-2026-10-30T17-51-10.172Z` at 17:58:09Z, with full parity (#1377 AC-5). Afterwards, one deployment-state snapshot held both of these:

- `maintenance.lastBackup`: `status: success`, the 17:51 bundle, restorable;
- `maintenance.retry`: `{phase: exhausted, retriesRemaining: 0, lastSuccessAt: 2026-10-28T13:58:02Z}`.

Native health reports `backup-retry-exhausted` and `backup-success-overdue`, and the FM view shows "backup lane: exhausted" in red (operator report, 22:0xZ). Captured first as defect-note d8da6e8c.

## The Problem

`lastSuccessAt` is written only by the scheduled task's `TaskStateService.markCompleted()`, so a CLI success never reaches the lane's ledger. The health derivation (`ai/daemons/orchestrator/scheduling/backup.mjs`, reason codes around lines 377/434) builds those two codes from the ledger. Its own comment says that when the ledger and a newer success receipt disagree, the receipt is the owner's record. The view stays red for up to a day after a good backup and keeps the operator guessing.

## The Fix

When a success receipt is stamped after the failure streak's anchor, the health derivation stops emitting `backup-success-overdue` and `backup-retry-exhausted`, or reports them as superseded. Do not make the CLI write task state: that would be a second producer of the ledger.

## Acceptance Criteria

- [ ] AC-1 A spec: a snapshot whose `lastBackup` success is newer than `retry.streakStartedAt` yields no `backup-success-overdue`/`backup-retry-exhausted` (or a superseded form), and the backup block's status is not degraded.
- [ ] AC-2 Without a newer receipt, the existing codes are unchanged (spec).
- [ ] AC-3 (installed) After a manual success, the FM's backup lane line no longer shows exhausted.

## Related

#1377, #969, defect-note d8da6e8c.

Live latest-open + closed sweep: checked the latest 20 open issues plus a search ("backup health retry exhausted overdue receipt") at 2026-10-30T22:5xZ; #969 is adjacent (off-host sync), no equivalent.

unowned-rationale: Ready for a Brain maintainer; the fix is a bounded rule plus spec.

Retrieval Hint: "backup health retry exhausted success overdue manual run receipt ledger"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f

## Timeline

- 2026-10-30T22:45:12Z @neo-opus-ada added the `bug` label
- 2026-10-30T22:45:12Z @neo-opus-ada added the `ai` label
- 2026-10-30T22:45:13Z @neo-opus-ada added the `agent-os` label
### @neo-opus-ada - 2026-10-31T15:01:34Z

**Datum, 2026-10-31: the live symptom cleared by itself; the defect stands.**

The scheduled task's attempt at 14:51:11Z succeeded. It was the first scheduled run since #1378 reached the plane (`4ca0f506`). Bundle `backup-2026-10-31T14-51-41.479Z` finished at 14:59:09Z with `status: success`, `restorable: true` and no empty subsystems. `markCompleted()` then wrote the ledger: `phase: healthy`, `lastSuccessAt: 2026-10-31T14:59:09.486Z`, next attempt in 24 h.

So the FM's "backup lane: exhausted" is gone today because the scheduled task itself succeeded, not because health now reads the receipt. The case this ticket is about is unchanged: a manual CLI success while the ledger is exhausted. To reproduce, run a manual backup during an exhausted streak; the ACs are spec-level and don't need that.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


