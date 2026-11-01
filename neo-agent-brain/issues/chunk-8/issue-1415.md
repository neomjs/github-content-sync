---
id: 1415
title: 'Backup retention keeps 7 days, not 30'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-01T02:43:18Z'
updatedAt: '2026-11-01T03:20:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1415'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-11-01T03:13:00Z'
---
# Backup retention keeps 7 days, not 30

## Context

On 2026-10-22, during the disk-full investigation, the operator asked to evaluate lowering the backup retention default from 30 days to 7. On 2026-11-01 he set it ("7 days default for us … must get a prio"). That drive is a 1.8 TB data volume at 99 %. `~/.neo-ai/backups` holds 33 bundles (Oct 2–31), 168.1 GB measured as private bytes, so deleting them frees exactly that. The 27 bundles older than 7 days hold 143.0 GB; the last 7 days hold 18.5 GB in 6. No ticket existed: #974 kept retention out of scope ("Retention (`maxDays`, `keepMinimum`) stays as is; it is a separate operator knob").

## The Problem

The 30-day default is written twice:

- the leaf: `maintenance.backup.retention.maxDays: 30` (`ai/configBase.mjs:2614`);
- a hidden default in `cleanOldBackups`, `const {keepMinimum = 3, maxDays = 30} = retention` (`ai/scripts/maintenance/backup.mjs:1322`), plus its JSDoc.

Production reads the leaf: `resolveBackupRetention()` (`backup.mjs:704`) feeds `cleanOldBackupsImpl(AiConfig.backupPath, logger, retention)` (`:562`). The function default serves only tests that pass no retention. A daily bundle is 3–9 GB, so a 30-day window holds 90–280 GB.

## The Architectural Reality

- `maintenance` is one object-default leaf (`leaf({…})`, `ai/configBase.mjs:2604`) with no env binding; an operator overlay overrides it.
- `detectionRetentionSlaInputs.mjs` reads `retention.maxDays` for the detection SLA (detect cadence at most half the retention). At 7 days the ceiling is 3.5 days against the 1 h integrity sweep, so it stays within SLA. Its docblock (`:19`) cites the function's default of 30.
- ADR 0019 (`adr:0019`): the leaf is the single source. A function default that restates it is a hidden default.
- Defrag snapshot retention is separate (`maintenance.defrag.snapshotRetention`, already 7 days).

## The Fix

1. `ai/configBase.mjs`: change `maxDays: 30` to `maxDays: 7`.
2. `ai/scripts/maintenance/backup.mjs`: `cleanOldBackups` drops its literal defaults. A field the caller leaves out falls back to the resolved leaf (`{...resolveBackupRetention(), ...retention}`), so pruning keeps running on a partial retention — the robustness `detectionRetentionSlaInputs.mjs:19` documents — with no second literal. Its JSDoc and the module docblock's Retention Policy name the leaf rather than literals.
3. `ai/scripts/maintenance/detectionRetentionSlaInputs.mjs`: correct the docblock line that cites the function default.
4. `backup-retention.spec.mjs`:
   - The default-config arm reads the shipped leaf through `resolveBackupRetention()`. It seeds bundles that tell 7 from 30: 1, 2, 3, 10 and 25 days leave 3 at 7 days, where 30 days would leave 5.
   - The two arms that exercised the function's literal defaults (`undefined`, `{}`) now prove the leaf fallback: a missing retention, and a partial one (`{keepMinimum: 3}`), both prune a 10-day bundle that 30 days would keep.
5. `detectionRetentionSlaInputs.spec.mjs`: its "today's shipped leaf values" become 7 days (`backupRetentionMs` 7 d, `requiredMaxDetectMs` 3.5 d).

## Acceptance Criteria

- [ ] AC-1: the canonical template resolves `maintenance.backup.retention.maxDays` to 7.
- [ ] AC-2: `cleanOldBackups` carries no literal retention default; a missing or partial retention falls back to the leaf per field.
- [ ] AC-3: under the shipped retention, a bundle aged 10–25 days outside the newest 3 is deleted. Red on base, where it survives at 30.
- [ ] AC-4: the detection SLA stays green at 7 days.

## Post-Merge Validation

- [ ] After a plane cut to a Brain carrying this, the next backup run leaves only bundles at most 7 days old plus the newest 3 restorable per substrate. The author posts the receipt here.

## Out of Scope

- Tiered retention (7 dailies plus weeklies). I recommended it on 2026-10-22 because a KB investigation needed an 11-day-old baseline; the operator chose a flat 7 days. It is a follow-up if a longer baseline is needed again.
- The off-host sync (#969).
- Deleting existing bundles by hand: the sweep does it on its first run after deploy.

## Decision Record impact

`aligned-with adr:0019`: the leaf stays the single source, and one hidden default is removed.

## Related

#974 (compression; retention out of scope) · neomjs/neo#11018 (orchestrator-owned daily backup with 30-rotation retention) · neomjs/neo#16203 (backups and the graph share one disk) · neomjs/neo#16614 (retention counts recoverability)

Live latest-open + closed sweep: latest 20 open + closed search ("backup retention"; #974's Out of Scope read) at 2026-11-01T02:42Z; no equivalent or prior exclusion. A2A last 30: no claim on this scope. Memory Core: the 2026-10-22 investigation. Own open assignments: no overlap.
Structure map: existing files only (`ai/configBase.mjs`, `ai/scripts/maintenance/`); no placement decision.
unowned-rationale: none, self-assigned (operator priority).

Origin Session ID: 4be090ce-62b5-49bd-a6e4-0a928e2fcbee
Retrieval Hint: "backup retention 7 days maxDays operator disk full"



## Timeline

- 2026-11-01T02:43:19Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-01T02:43:20Z @neo-opus-ada added the `enhancement` label
- 2026-11-01T02:43:20Z @neo-opus-ada added the `ai` label
- 2026-11-01T02:43:21Z @neo-opus-ada added the `agent-os` label
- 2026-11-01T02:47:27Z @neo-fable-mnemosyne cross-referenced by #1416
- 2026-11-01T02:50:34Z @neo-opus-ada cross-referenced by PR #1417
- 2026-11-01T03:02:49Z @neo-opus-ada referenced in commit `9b0c38e` - "fix(backup): the config template spec expects the 7-day retention (#1415)

The template spec pins Config.maintenance.backup with toEqual, so it carried the old 30-day value. The leaf now resolves 7."
- 2026-11-01T03:13:00Z @tobiu referenced in commit `804908b` - "fix(backup): retention keeps 7 days, not 30 (#1415) (#1417)

* fix(backup): retention keeps 7 days, not 30 (#1415)

The operator set a 7-day backup retention default. At 30 days the local
plane kept 33 bundles (168 GB) on a 99%-full drive.

The leaf maintenance.backup.retention.maxDays drops from 30 to 7.
cleanOldBackups no longer restates the policy as literal defaults: a
field the caller leaves out falls back to the resolved leaf, so pruning
still runs on a partial retention. The detection SLA stays green
(ceiling 3.5 d against the 1 h integrity sweep).

* fix(backup): the config template spec expects the 7-day retention (#1415)

The template spec pins Config.maintenance.backup with toEqual, so it carried the old 30-day value. The leaf now resolves 7."
- 2026-11-01T03:13:02Z @tobiu closed this issue
### @neo-opus-ada - 2026-11-01T03:13:57Z

**Merged; the deploy rides the next plane cut, at the latest right after the YC cutoff (Tue Nov 3, 05:00 CET).**

PR #1417 merged at 2026-11-01T03:12:59Z as `804908b7`. The live plane still runs `63e99c65`, so the 7-day default takes effect with the next plane cut.

Why not tonight:
- A plane cut takes Memory Core, KB and the orchestrator down for every seat, and the seats are mid-push on the YC field.
- Nothing waits on the sweep. The operator moved the 27 bundles older than 7 days and the 2 aborted partials out by hand at about 02:52Z, which leaves 6 bundles (Oct 25–31, 23 GB).
- Until the cut, the 30-day policy lets roughly one more 3 GB bundle a day accumulate.

Post-Merge Validation (above) stays open: after the cut, the first backup's sweep must leave only bundles at most 7 days old plus the newest 3 restorable. I'll post that receipt here.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-11-01T03:20:16Z

**Deployed: the plane runs `804908b7` since 03:19:31Z.** The operator asked for the cut now, which supersedes the deferral above.

- `cut.sh` receipts: preflight `PROCEED_VERIFIED` (a restorable pre-cut bundle, 145,012 rows); the four services healthy at image revision and `/app/.neo-revision` `804908b739017edad3682e84f49bc4633f4ef5d0`; Chroma and ingress untouched.
- Live-code check in the running orchestrator: `backup.mjs` carries `{...resolveBackupRetention(), ...retention}` (1 match), and `ai/configBase.mjs:2614` reads `maxDays    : 7,`.
- The MC healthcheck reports `deployedRevision` 804908b7, with backup healthy and restorable (last successful 2026-10-31T14-51-41Z).

Post-Merge Validation stays open until the first backup on this revision. Its sweep must leave only bundles at most 7 days old plus the newest 3 restorable. The operator already moved the older ones out by hand, so expect the receipt to show the window holding rather than a mass prune. I'll post it here after that run.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


