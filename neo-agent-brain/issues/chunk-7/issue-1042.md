---
id: 1042
title: Renderer and marker waits can report timeout instead of their refusal
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-gemini-pro
createdAt: '2026-10-15T09:38:00Z'
updatedAt: '2026-10-15T10:34:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1042'
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
closedAt: '2026-10-15T10:34:59Z'
---
# Renderer and marker waits can report timeout instead of their refusal

## Context

#1032 fixed the endpoint wait in `ai/services/fleet/antigravityProjectBinding.mjs`. Its PR, #1039, adds `waitForPollInterval`, a sleep that resolves at the poll delay and rejects only on abort. #1032's scope excluded the binder steps after the endpoint proof, and two of those steps still sleep the old way.

## The Problem

At #1039's head `f86fbbd8`, `antigravityProjectBinding.spec.mjs` › "an accepted future build still refuses route mismatch and a missing project-selector marker" failed 4 of its 400 runs, about 1 %, in each of two `-g marker --repeat-each=400 --workers=24` runs (three tests, 1,200 runs per run). Every failure was `Expected "marker-mismatch"`, `Received "timeout"`. The mechanism is #1032's: on the final sleep, `within(sleep)`'s deadline timer and the sleep target the same instant, and the timer can fire first. As in #1032, a failure that occurs only on head turns an unrelated Brain PR red.

## The Architectural Reality

- `waitForUniqueRenderer` (head :347) and `waitForProjectMarker` (head :507) both sleep through `within(new Promise(resolve => setTimeout(resolve, Math.min(POLL_INTERVAL, remaining(deadline)))), deadline, signal)`. After the loop they throw `renderer-unproven` and `marker-mismatch`.
- `waitForPollInterval(delay, signal)` from #1039 is the sleep without the race.

## The Fix

Give both loops the shape #1039 gives the endpoint wait. They sleep through `waitForPollInterval`, break when the delay reaches 0, and check abort before their terminal refusal.

## Acceptance Criteria

- [ ] AC-1: when the deadline lands on the final sleep, `waitForUniqueRenderer` and `waitForProjectMarker` report their terminal refusal, whatever the timer order; abort during the sleep still refuses `canceled`.
- [ ] AC-2: PR receipt: the marker test above at `--repeat-each=400 --workers=24` shows 0 failures.

## Out of Scope

CDP steps still in flight at the deadline, whose `timeout` stays; the endpoint wait (#1032).

## Related

#1032 · #1039 (the helper this depends on) · #1017

handoff: @neo-gemini-pro (the author of #1039; proposed, not assigned)

Sweeps at 2026-10-15T09:45Z: the latest open Brain issues and owner-wide searches for `marker-mismatch` and `waitForProjectMarker` found no equivalent. Placement: an existing file, so no new placement. Decision Record impact: none.

Retrieval Hint: "Antigravity marker renderer wait final sleep timeout marker-mismatch timer order"


## Timeline

- 2026-10-15T09:38:02Z @neo-opus-ada added the `bug` label
- 2026-10-15T09:38:02Z @neo-opus-ada added the `ai` label
- 2026-10-15T09:39:45Z @neo-opus-ada cross-referenced by PR #1039
- 2026-10-15T10:10:57Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-15T10:15:45Z @neo-gemini-pro cross-referenced by PR #1045
### @neo-opus-ada - 2026-10-15T10:21:35Z

CI sighting, 2026-10-15 ~10:20Z. The `unit` job of #1040 ([run 38718640969](https://github.com/neomjs/neo-agent-brain/actions/runs/38718640969)) failed only on `antigravityProjectBinding.spec.mjs` › "an accepted future build still refuses route mismatch and a missing project-selector marker", with `"code": "timeout"` where `"marker-mismatch"` was expected. #1040 touches only `ai/services/github-workflow/**` and its specs. That is this ticket's Problem in practice: the head-only failure turned an unrelated PR red. I re-ran the failed job.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-15T10:35:00Z @tobiu referenced in commit `7bfe887` - "fix(fleet): report terminal refusal when renderer or marker wait expires on final sleep (#1042) (#1045)"
- 2026-10-15T10:35:00Z @tobiu closed this issue
- 2026-10-15T10:37:57Z @neo-opus-ada cross-referenced by #1038
- 2026-10-15T11:09:24Z @neo-opus-vega cross-referenced by #1046

