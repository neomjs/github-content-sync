---
id: 717
title: 'The receiver spec asserts which trigger adopted a publish, so a fast sweep fails it under load'
state: CLOSED
labels: []
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T18:39:06Z'
updatedAt: '2026-10-01T19:19:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/717'
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
closedAt: '2026-10-01T19:19:32Z'
---
# The receiver spec asserts which trigger adopted a publish, so a fast sweep fails it under load

## Context

Clio's defect-note (A2A, 2026-10-01 18:00Z) reported a failing arm. `test/playwright/unit/ai/daemons/wake/receiver.spec.mjs` "a distinct publish sharing an mtime is still adopted" failed on PR head suites (#706 twice, #707's first run) while the base suites passed. The differential gate (#650) then blocks each of those PRs, reporting the failure as "introduced".

## The Problem

The arm's receiver (`describe 'manifest reload'`) runs both its manifest watcher and a 40 ms sweep (`reconcileIntervalMs: 40`). After the second publish, the arm asserts that its own explicit `reloadIfChanged()` returns `1`. Under load, the watcher or the sweep can adopt the publish first. The explicit call then correctly reports "unchanged" (`null`), and the arm fails even though the receiver behaved correctly.

## The Fix

Assert the outcome instead of the trigger: after the second publish, the new route is served and the old one is not, whichever trigger adopted it. Every trigger compares the same revision identity, `mtime:size:inode` (`ai/daemons/wake/receiver.mjs:665`). So if that identity aliased the two publishes, the arm would still fail.

## Acceptance Criteria

- [ ] The arm asserts adoption through the served route table, not through the explicit call's return value.
- [ ] Red control: with the inode term removed from the revision identity, the arm fails.
- [ ] Race control: when the sweep is forced to win (a delay before the explicit call), the old assertion fails and the new one passes.

## Out of Scope

- `ai/daemons/orchestrator/daemon.spec.mjs:38` ("a DEAD predecessor is waited out"), the note's second arm. It belongs to the lease-succession owner.

## Related

#650 (the differential gate) · #706 / #707 (the PRs it hit) · #201 (the base's own reds)

Live latest-open sweep at 2026-10-01T18:4xZ covered the 15 most recent open Brain issues, plus keyword searches for "receiver.spec flaky" and "mtime adopted" (Brain) and "flaky head base comparator introduced" (org). No equivalent found.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207


## Timeline

- 2026-10-01T18:39:07Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T18:40:40Z @neo-opus-vega referenced in commit `28fde31` - "fix(wake): the receiver spec asserts that a same-mtime publish is served, not which trigger adopted it (#717)

The arm's receiver runs its watcher and a 40 ms sweep, and the arm required
its own explicit reloadIfChanged() to report the adoption. Under load the
sweep adopts first, the explicit call correctly answers "unchanged", and the
arm went red on PR head suites, which the differential gate counts as
introduced. The arm now waits for the served table: the new route answers,
the old one is gone. Every trigger compares the same mtime:size:inode
revision, so an identity that aliased the two publishes still fails it."
- 2026-10-01T18:41:02Z @neo-opus-vega cross-referenced by PR #718
- 2026-10-01T18:42:59Z @neo-opus-vega cross-referenced by #719
- 2026-10-01T19:17:00Z @neo-opus-ada cross-referenced by PR #724
- 2026-10-01T19:19:32Z @tobiu referenced in commit `b5aa667` - "fix(wake): the receiver spec asserts that a same-mtime publish is served, not which trigger adopted it (#717) (#718)

The arm's receiver runs its watcher and a 40 ms sweep, and the arm required
its own explicit reloadIfChanged() to report the adoption. Under load the
sweep adopts first, the explicit call correctly answers "unchanged", and the
arm went red on PR head suites, which the differential gate counts as
introduced. The arm now waits for the served table: the new route answers,
the old one is gone. Every trigger compares the same mtime:size:inode
revision, so an identity that aliased the two publishes still fails it."
- 2026-10-01T19:19:33Z @tobiu closed this issue
- 2026-10-01T19:31:34Z @neo-opus-grace cross-referenced by #725

