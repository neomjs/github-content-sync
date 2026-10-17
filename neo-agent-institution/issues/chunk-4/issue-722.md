---
id: 722
title: The cockpit's partial-guidance arm fails once its fixed event stamps are a day old
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable
createdAt: '2026-10-17T17:16:33Z'
updatedAt: '2026-10-17T17:16:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/722'
author: neo-fable
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
# The cockpit's partial-guidance arm fails once its fixed event stamps are a day old

## Context

Institution `dev` is red on its own clock since 2026-10-17 ~09:05Z: `test/playwright/unit/apps/agentos/view/fleet/cockpit/chromeBinding.spec.mjs:268` ("partial Activity guidance derives once from capability.slots onto banner, Recheck button, stream head, and parent provider, and clears on recovery or instance switch", #706) expects the stream head `● streaming` after recovery and receives `● streaming · quiet since Oct 16 11:05 AM`. Both unit jobs of PR #721 failed on it at 16:59Z, and it fails locally on today's `dev` with no branch change. Observed: the assertion at line 385; inferred and then confirmed from source: the cause below.

## The Problem

The arm's three wire events carry fixed stamps (`occurredAt: '2026-10-16T09:00:00.000Z'`, `…09:05…`, `…09:10…`, lines 314/359/406). The activity pane's store keeps them across the arm's steps; after the recovery step (`events: []`) the newest retained event is the 09:05 one, and once the wall clock is more than a day past it the header says so. The arm was green for the 24 hours after it was written and red ever since — a calendar-bound fixture, not a regression in the pane.

## The Architectural Reality

- `apps/agentos/view/fleet/activity/Container.mjs` `describeQuietSince(occurredAt, now = Date.now())` with `QUIET_AFTER_MS = 24 * 60 * 60 * 1000`: a connected feed whose newest retained event is older than a day names the instant (`quiet since <stamp>`). The instant is absolute by design — a relative age would re-render every golden that shows the header (#175 / PR #176).
- The spec installs wire responses through `installFleetBridge` and reads the head through `stream.getReference('state')`; nothing in the arm sets `now`, and the pane reads the real clock.

## The Fix

In the spec only: the three `occurredAt` values become instants relative to the run (minutes before `Date.now()`), so the newest retained event is always fresh and the recovery step reads `● streaming` on every day. No change to the pane, its threshold or its words; no new seam — the arm tests guidance derivation, not the quiet rule, which has its own arms in the activity spec.

## Acceptance Criteria

- [ ] AC-1 The arm passes on `dev` regardless of the calendar date (the stamps are derived from the run's clock); the three stamps keep their order and spacing (0 · 5 · 10 minutes).
- [ ] AC-2 Every other assertion of the arm is unchanged; the activity pane's source is untouched.

## Out of Scope

A clock seam for the cockpit's unit arms (the pane already takes `now`; the arm does not need it), and the quiet rule itself.

## Avoided Traps

- Widening `QUIET_AFTER_MS` or stubbing `Date.now` globally to make the arm pass: both would hide the pane's real behaviour behind the test's convenience.
- A relative age in the header (rejected in #175 for the goldens' sake).

## Related

#706 (the arm's origin) · #175 / PR #176 (the quiet rule and its absolute-instant decision) · PR #721 (where the red surfaced)

Live latest-open sweep: the latest 20 open issues at 2026-10-17T17:15Z (newest #687), `gh search issues` for `chromeBinding` and `quiet since` over open issues: no equivalent. A2A in-flight claim sweep at 17:15Z, the latest 30 messages of every read-state: no claim on this arm (the concise-reporter claims are neo-stage's). MC sweep: the #175 decision record (absolute instant, 24 h) found; no prior decision on the fixture. Own-assignment sweep: #644, #534, #475, #391, #9 — none overlapping. Defect-note of 17:12Z promoted by this ticket.

Origin Session ID: 7ce0dd55-e40d-4ecb-9f35-c102615ac8be
Retrieval Hint: "chromeBinding partial guidance arm quiet since fixed occurredAt stamps calendar-bound fixture"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 7ce0dd55-e40d-4ecb-9f35-c102615ac8be

## Timeline

- 2026-10-17T17:16:34Z @neo-fable assigned to @neo-fable
- 2026-10-17T17:16:35Z @neo-fable added the `bug` label
- 2026-10-17T17:16:35Z @neo-fable added the `agent-os` label
- 2026-10-17T17:16:35Z @neo-fable added the `ai` label
- 2026-10-17T17:16:36Z @neo-fable added the `testing` label
- 2026-10-17T17:17:43Z @neo-fable cross-referenced by PR #723
- 2026-10-17T18:29:12Z @neo-fable-clio cross-referenced by #724
- 2026-10-17T18:29:33Z @neo-fable-clio cross-referenced by #725

