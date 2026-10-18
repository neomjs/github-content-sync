---
id: 1133
title: daemon.spec's stale-replay arm races a fixed 250 ms sleep against the suppression log and the watermark write
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T08:00:55Z'
updatedAt: '2026-10-18T14:45:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1133'
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
closedAt: '2026-10-18T14:45:45Z'
---
# daemon.spec's stale-replay arm races a fixed 250 ms sleep against the suppression log and the watermark write

## Context

`test/playwright/unit/ai/daemons/wake/daemon.spec.mjs` › `#15704: stale replay stays unread and consumed while a fresh peer alone shapes the digest` failed in CI on two of three attempts of run 39107522011 (PR #1126), and compareUnitFailures counted it as "introduced". #1126 changes nothing the wake daemon reaches. The arm passes 10/10 locally at the same head.

## The Problem

After the fresh delivery, the arm sleeps a fixed 250 ms ("watermark persistence follows the delivery log"), then reads `wake-daemon.log` and `woken-watermark.json` once. The daemon writes the stale-suppression line and persists the watermark after the dispatch. In both failed attempts the log held the delivery and dispatch lines but not `Suppressed 1 stale/invalid message wake event(s) … at initial delivery`, so the window expired under CI load first. Any change that reshuffles which specs share a worker can surface the race, and #1126 did.

## The Architectural Reality

The same spec already waits on observed state elsewhere, with `expect.poll(…, {timeout: 10000})`. Only this arm and its siblings still sleep a fixed time.

## The Fix

Replace the arm's 250 ms sleep with polls on the observed state: the log contains the suppression line, and the durable watermark lists both message ids. The read-after assertions then run on settled state.

## Acceptance Criteria

- [ ] **AC-1:** the arm passes at head, locally and in CI. Its assertions are unchanged; only the wait changes.
- [ ] **AC-2:** the arm holds under contention. It passes under parallel CPU load where the 250 ms version fails, or, if load cannot reproduce the race, a delayed suppression line shows the old version failing and the new one waiting.

## Out of Scope

- The spec's other fixed sleeps, at the arms around lines 460–525, 1076–1078, 1323 and 1782–1977. They are the same anti-pattern, with no observed failure yet.
- The `DatabaseService.backup` arm that failed attempt 3 (`source-collection-empty` vs `source-collection-unresolved`). It was seen once and is noted separately.

## Related

- #201: the CI baseline this keeps honest.
- #1126: where it surfaced.

Live latest-open sweep: the latest 20 open Brain issues, rechecked at 2026-10-18T08:05Z; no equivalent found. `gh search` for "15704" and "daemon.spec flaky" found no ticket.
A2A claim sweep: the latest inbox messages; my own defect-note (07:51) is the only mention.
Own-assignment sweep: #201 and the open #201 slices.
Structure map: N/A; this changes one spec arm.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "wake daemon.spec 15704 stale replay fixed 250ms sleep race suppression log watermark expect.poll"

## Timeline

- 2026-10-18T08:00:55Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T08:00:57Z @neo-opus-ada added the `bug` label
- 2026-10-18T08:00:57Z @neo-opus-ada added the `ai` label
- 2026-10-18T08:00:57Z @neo-opus-ada added the `testing` label
- 2026-10-18T08:07:10Z @neo-opus-ada cross-referenced by PR #1134
- 2026-10-18T08:07:24Z @neo-opus-ada cross-referenced by PR #1126
- 2026-10-18T14:45:45Z @tobiu closed this issue
- 2026-10-18T14:45:45Z @tobiu referenced in commit `d6a905d` - "fix(test): the stale-replay arm waits on the suppression line and the watermark, not 250 ms (#1133) (#1134)

Both the suppression line and the durable watermark are written after the delivery the arm awaited, so a fixed 250 ms window lost the race under CI contention (2 of 3 attempts on PR #1126). The arm now polls the observed state and keeps every assertion."

