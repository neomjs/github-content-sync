---
id: 261
title: The packaged smoke waits 20 s for a roster read now due every 60 s
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T21:04:28Z'
updatedAt: '2026-09-26T22:14:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/261'
author: neo-opus-grace
commentsCount: 0
parentIssue: 7
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T22:14:30Z'
---
# The packaged smoke waits 20 s for a roster read now due every 60 s

## Context

Emmy validated the installed app built from Institution `9bd959a9` after `#257` (Resolves `#255`) merged. In the packaged shell smoke, `workerAfterPopupClose` fails. `firstPaint`, IPC and `sharedHeap` pass (A2A, 2026-09-26 20:57Z). `#257` gave each cockpit liveness read its own cadence. The roster read now comes due every 60 s (`LivenessCadence.DEFAULT_INTERVALS.roster`), and the cockpit's 15 s pass launches it (`livenessPollDefault`, `apps/agentos/view/fleet/cockpit/Container.mjs:30`).

## The Problem

The Brain leg of `harness/main.mjs` closes the popup, then waits 20 s for one more `fleetRoster` call: "a fresh App-Worker crossing after popup close". 20 s fit the old shared 15 s tick. Under the per-read cadence, the next roster read can be up to 75 s away (60 s interval plus one 15 s pass). The check therefore passes only when the close happens to land shortly before a scheduled read. Nothing tied the smoke's window to the cockpit's cadence. `#257`'s CI stayed green because no CI job runs the harness smoke (see `#214`).

## The Architectural Reality

- `harness/adapterWitness.mjs` holds the cockpit contract that the smoke's verdict relies on. It was extracted so that half is unit-testable without Electron, and `test/playwright/unit/harness/adapterWitness.spec.mjs` covers it. The harness cannot import the cockpit's Neo classes; Neo imports stay in thread entrypoints.
- `apps/agentos/util/LivenessCadence.mjs` owns the read intervals. The pass default is an unexported module const in `Container.mjs`.
- `firstWorkerCrossing`, the roster read at cockpit start, is unaffected.

## The Fix

1. Move the pass default into `LivenessCadence` beside the intervals, so the cockpit's cadence has one importable home. `Container` reads it from there.
2. Export a roster-crossing window from `adapterWitness.mjs`, sized to one roster interval plus one pass plus a margin. `main.mjs` waits that long for the crossing after the popup closes.
3. Add an `adapterWitness.spec.mjs` arm that fails when the cockpit's roster interval plus pass outgrows the window.

## Acceptance Criteria

- [ ] AC-1: `workerAfterPopupClose` waits for the window exported by `adapterWitness.mjs`, not a literal in `main.mjs`, and the window covers one roster interval plus one pass.
- [ ] AC-2: a unit arm goes red when `LivenessCadence`'s roster interval plus pass exceeds the window (mutation-checked).
- [ ] AC-3 (post-merge): a packaged smoke run reports `workerAfterPopupClose: true`.

## Out of Scope

- Forcing a cockpit read from the shell. No product seam exists for it, and the check proves that the scheduled crossing still happens after a popup closes.
- A CI job for the harness smoke; `#214`'s reasons stand.

## Avoided Traps

- Raising the literal in `main.mjs` alone repeats the defect: the next cadence change breaks the smoke silently again.

## Related

Regression from #257 (#255). Parent epic: #7. Related: #214, #259.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-26T21:03Z hold no equivalent (newest `#259`). A2A in-flight sweep: the latest 10 messages across all read-states, 20:36–21:02Z, show no competing claim; Emmy routed the failure to me. Memory Core problem-noun sweep: no prior decision on the smoke window. Own assignments: `#258` and `#11` do not overlap.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "packaged smoke workerAfterPopupClose roster cadence window"

## Timeline

- 2026-09-26T21:04:29Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T21:04:30Z @neo-opus-grace added the `bug` label
- 2026-09-26T21:04:30Z @neo-opus-grace added the `ai` label
- 2026-09-26T21:04:30Z @neo-opus-grace added the `regression` label
- 2026-09-26T21:04:30Z @neo-opus-grace added the `testing` label
- 2026-09-26T21:04:50Z @neo-opus-grace added parent issue #7
- 2026-09-26T21:09:53Z @neo-opus-grace cross-referenced by PR #262
- 2026-09-26T21:11:21Z @neo-opus-grace cross-referenced by #10
- 2026-09-26T22:06:46Z @neo-opus-grace cross-referenced by #264
- 2026-09-26T22:14:30Z @tobiu referenced in commit `1acb7ec` - "Merge pull request #262 from neomjs/grace/261-smoke-roster-window

fix(harness): the smoke waits a roster interval plus a pass for the worker crossing (#261)"
- 2026-09-26T22:14:30Z @tobiu closed this issue
- 2026-09-26T22:32:45Z @neo-gpt-emmy cross-referenced by #7

