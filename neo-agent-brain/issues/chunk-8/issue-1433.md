---
id: 1433
title: 'The data-integrity sweep never completes: its lane has no state envelope'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-02T17:34:38Z'
updatedAt: '2026-11-02T18:10:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1433'
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
blocking:
  - '[ ] 1434 No producer measures the KB''s dead HNSW elements on the cloud plane'
closedAt: '2026-11-02T18:01:05Z'
---
# The data-integrity sweep never completes: its lane has no state envelope

## Context

While measuring #1432's detect half on this plane (Brain `59c5a551`), I found that the orchestrator's state file holds no `data-integrity-sweep` entry, although the lane is registered and due every hour. Its siblings persist their runs: `embed-drain-liveness-watchdog` ran at 16:37:11Z and `heavy-maintenance-starvation-watchdog` at 17:22:00Z (`/app/.neo-ai-data/orchestrator-daemon/orchestrator-state.json`, 25 keys).

## The Problem

The sweep has no task-state envelope. Its runner throws before it records anything, and its own error path throws the same way. Reproduced at `59c5a551` by building the real table (`buildTaskDefinitions()` → `createInitialTaskState()`) and calling the two `TaskStateService` writers the runner uses:

```
registry tasks: 20 | state keys: 24
registry tasks with NO state envelope: [ 'data-integrity-sweep' ]
markStarted throws: TypeError - Cannot set properties of undefined (setting 'running')
markFailed throws: TypeError - Cannot set properties of undefined (setting 'running')
```

- `runDataIntegritySweepTask` (`scheduling/pipeline.mjs` l.1245) calls `markStarted` first, which throws.
- Its catch calls `markFailed`, which throws again, and the last-resort `catch {}` swallows that. The `recordTaskOutcome` and `writeLog` calls after it never run: no health record, no log line.
- `lastRunAt` is never written, so `getDueTask` (`scheduling/dataIntegritySweep.mjs` l.27) reports the lane due on every poll. The pipeline dispatches it on every poll, and it dies silently every time.
- neo `#14117` (2026-06-26, for `#14109`, my PR) scheduled the runner without a `taskDefinitions.mjs` entry, and no later commit added one. `markStarted` dereferenced unguarded at `#14117`, at `#16903` and at Brain `#415`, as it does today. So the sweep has never completed a run on any plane, and the live loop from ADR 0025's detect half to ADR 0027's data actuator (coverage → diagnosis → heal) has been dark since June.
- CI stayed green: `pipeline.spec.mjs` (l.1398ff) drives the runner against a recording `taskStateService` mock, and no spec checks that each registry task has an envelope.

## The Architectural Reality

- `TaskStateService` builds its envelopes from the task-definition table alone (`createInitialTaskState`, `services/TaskStateService.mjs` l.9). The orchestrator hands it the authority-scoped table (`Orchestrator.mjs` l.481). Its writers dereference the entry unguarded (`markStarted` l.265, `markFailed` l.420).
- The scheduler walks a different table, `TASK_REGISTRY` (`scheduling/registry.mjs`). The in-process health checks bridge the two with inert definition entries: the `embed-drain-liveness-watchdog` entry "exists only so the task gets a persisted state envelope" (`taskDefinitions.mjs` l.510–537). Three siblings have one; the sweep does not.
- **The first live run fails closed.** The orchestrator image ships no `sqlite3` binary and does not mount `chroma-data` (`command -v sqlite3` finds nothing; `/app/.neo-ai-data/chroma` is absent). The coverage gatherer (`Orchestrator.mjs` l.1378) shells out to `sqlite3`, so `gatherAndDiagnose` returns `probe-unavailable` and actions nothing (`services/DataIntegrityDiagnosisService.mjs` l.117–125). On this plane the fix turns a silent lane into an honest one and fires no heal.
- Design authority: the registry entry's own comment, "The data-integrity sweep: live scheduling of the data-integrity DETECT signal" (`scheduling/registry.mjs` l.403), and the runner's never-fail contract, "record the fault and clear running state" (`scheduling/pipeline.mjs` l.1276), which it cannot keep without an envelope.
- Placement: `ai/daemons/orchestrator/taskDefinitions.mjs` (370 LOC) holds the three sibling entries (`npm run ai:structure-map -- --files --loc`).

## The Fix

1. Add a `data-integrity-sweep` entry to `buildTaskDefinitions` in `taskDefinitions.mjs`, beside its three siblings and with their inert posture.
2. Add an invariant spec: under the container-plane authority profile, every `TASK_REGISTRY` task has an envelope in `createInitialTaskState(<authority-scoped definitions>)`. Drift between the two tables then fails CI.

## Acceptance Criteria

- [x] AC-1 `createInitialTaskState(buildTaskDefinitions())` holds a `data-integrity-sweep` envelope.
- [x] AC-2 A spec fails when any container-plane `TASK_REGISTRY` task lacks an envelope. It fails at `59c5a551` and passes with the fix.
- [x] AC-3 *(post-merge, live; met 18:08:33Z, [receipt](https://github.com/neomjs/neo-agent-brain/issues/1433#issuecomment-6584625776): the outcome is reproduced, because the decision is held in-process only)* After the next plane cut, `orchestrator-state.json` holds a `data-integrity-sweep` entry with a `lastRunAt` inside the hour, and the outcome is recorded (expected: `probe-unavailable` naming the gatherer's error). Posted on this ticket.

## Out of Scope

- Giving the gatherer a store to read, and the dead-element producer: the detect leaf for #1432.
- Hardening every health-check runner's catch. The invariant spec closes this path; a `writeState` fault inside the catch is a separate class with no sighting.

## Avoided Traps

- **Deriving envelopes from `TASK_REGISTRY` by construction.** It would retire the three inert entries, but the process supervisor, the backpressure service and the deployment-state bridge all read the same state map. Widening its keys carries its own blast radius, while the invariant spec closes the drift now.
- **Guarding `markStarted` against a missing entry.** That would hide the next missing envelope the way the catch hid this one.

## Related

- #1432: the KB defrag op whose detect half runs inside this sweep. #1416 is the parent investigation.
- ADR 0025 (detect + diagnose) and ADR 0027 (the data-world actuator): this sweep is the live loop between them.
- Introduced by neo `#14117` (`#14109`).

**Sweeps, 2026-11-02 17:33Z:**
- Live latest-open + closed sweep: checked the latest 20 open issues and a closed search for `data-integrity sweep envelope` (hits #1430 and #12, neither on this surface). No equivalent or prior exclusion found.
- A2A in-flight sweep: the last 30 messages carry no claim on this lane.
- MC sweep: `data-integrity sweep never runs, no state persisted, orchestrator health check silent`, 6 results. They show the June wiring (neo `#14117`) and Emmy's 09-20 read, which assumed an hourly sweep. No prior decision found.
- Own-assignment sweep: 14 open. #1432 consumes this fix; none overlaps.

Ownership: self-assigned; it blocks #1432's detect leaf, which I own.

Retrieval Hint: "data-integrity-sweep task-state envelope markStarted TypeError swallowed never completes"



## Timeline

- 2026-11-02T17:34:39Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-02T17:34:40Z @neo-opus-vega added the `bug` label
- 2026-11-02T17:34:40Z @neo-opus-vega added the `ai` label
- 2026-11-02T17:34:41Z @neo-opus-vega added the `agent-os` label
- 2026-11-02T17:37:30Z @neo-opus-vega cross-referenced by #1434
- 2026-11-02T17:39:50Z @neo-opus-vega cross-referenced by #1432
- 2026-11-02T17:46:07Z @neo-opus-vega cross-referenced by PR #1436
- 2026-11-02T17:48:26Z @neo-opus-vega cross-referenced by #1416
- 2026-11-02T18:01:05Z @tobiu referenced in commit `92a7ec4` - "fix(orchestrator): give the data-integrity sweep its task-state envelope (#1433) (#1436)

The sweep is scheduled from TASK_REGISTRY but had no entry in the
task-definition table, so TaskStateService gave it no envelope. markStarted
threw on every poll, the runner's catch threw again in markFailed, and the
last-resort catch swallowed both: no state, no health record and no log line
since neomjs/neo#14117. Add the inert entry its three in-process siblings
carry, plus an invariant spec that fails when any registry lane lacks an
envelope. Orchestrator.spec.mjs pins the exact key list, so it gains the lane."
- 2026-11-02T18:01:06Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-02T18:10:08Z

**AC-3, live receipt: the sweep completed its first run ever at 18:08:33Z.** This is the plane cut to `92a7ec44` (#1436). Cut 18:06:46–18:08:37Z; all four services healthy at the SHA; chroma and ingress untouched.

`orchestrator-state.json` now holds 26 keys, including:
```
data-integrity-sweep: lastRunAt 1793642913981 · lastCompletionAt 2026-11-02T18:08:33.989Z · lastExitCode 0 · lastReason periodic-data-integrity-check:3600000
```
That is the first poll after the orchestrator started (18:08:24Z). The runner no longer throws, and its state is persisted.

**The outcome is `probe-unavailable`, reproduced rather than read.** The decision is held only in the orchestrator's in-process `HealthService` (`#taskOutcomes`): no file, endpoint or log line exposes it, so I reproduced the coverage reader's call inside the container:
```
chroma dataDir present: false
coverage reader probe: spawn sqlite3 ENOENT
```
So the sweep records `probe-unavailable` hourly and actions nothing. It also pages nobody: only `backup` is in `ESCALATING_TASK_OUTCOMES`. #1434 gives it a store to read.

Sighting: the decision's in-memory-only state is a gap. It is filed as a defect-note, not a ticket.

— Vega (Claude Opus 5.5, Claude Code) 🌿

- 2026-11-02T18:14:15Z @neo-opus-vega cross-referenced by #15000

