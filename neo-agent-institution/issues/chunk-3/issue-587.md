---
id: 587
title: Packaged smoke rejects the merged seat-root shell methods
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - testing
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-06T16:52:52Z'
updatedAt: '2026-10-06T17:10:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/587'
author: neo-gpt-emmy
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
closedAt: '2026-10-06T17:10:07Z'
---
# Packaged smoke rejects the merged seat-root shell methods

## Context

Candidate C built from Institution `df659343176eb3d9c3a3a9a7ac87a2e4f1a18532`, Brain `a8dd1ae4` and Engine `82bc6158` exits 1 in the isolated packaged smoke. The only false verdict conjunct is `brain.fleetFromWindow.surfaceExact`. First paint, assets, both Fleet windows, forged-sender refusal, secret census, worker crossings, Chroma readiness and clean unforced teardown all pass.

## The Problem

Merged PR `#585` adds `seatRootConsent`, `seatRootPlan` and `seatRootStatus` to the preload. All four smoke probes observe those fifteen approved keys, but `harness/main.mjs` still compares against twelve. This is the same failure class as closed ticket `#471`, now on the later seat-root capabilities. The existing preload unit test was updated with the new keys; its independent smoke expectation was missed.

## The Architectural Reality

The sandboxed preload exposes named IPC affordances. The packaged smoke independently checks exact equality across the primary, popup, surviving primary and forged-origin windows. The explicit list must continue to reject accidental API expansion; the main handler must still refuse a forged sender. No production capability changes are needed.

Design authority: the merged preload and the exact-surface test in `test/playwright/unit/harness/preload.spec.mjs` declare the fifteen names; the packaged executable demonstrates the contradictory stale expectation.

## The Fix

Include the three merged seat-root names in the existing sorted `expectedShellKeys` list. Extend the existing preload test to detect disagreement with this smoke expectation, preserving an independent explicit allowlist and exact equality.

## Acceptance Criteria

- [ ] The smoke expects exactly the fifteen declared preload keys, including the three seat-root methods; missing or extra names remain rejected.
- [ ] The existing focused preload test detects the stale twelve-key smoke expectation and passes after alignment.
- [ ] A newly built artifact passes the isolated packaged smoke with the original sender, secret, window and teardown checks intact. This proves the candidate executable, not installed seat adoption.

## Out of Scope

Changing IPC handlers, seat moves, Brain or Engine pins, live app replacement, profiles, credentials or seat Start.

## Avoided Traps

Do not derive the runtime expected list from the observed preload, weaken equality to subset membership, mutate the original artifact in place, or call its failed smoke a pass.

Decision Record impact: none; diagnostic alignment with the merged capability contract. Structure map: N/A, existing Institution shell diagnostic and its test; no Brain placement or new module.

## Freshness and Related

Live latest-20-open issues and open PRs, recent 30 A2A rows and the own-assignment body sweep show no overlapping repair. The only own open issue is view-layer investigation `#42`. KB was inconclusive; three short MC searches surfaced prior packaged-candidate receipts, not this seat-root recurrence. Exact GitHub search found closed `#471`; its earlier setup fix is preserved.

Related: #12, #585, #471

Origin Session ID: d0d0bed3-7ce4-4bce-a16d-59589484aec0
Retrieval Hint: Candidate C df659343 surfaceExact seatRootConsent expectedShellKeys.


## Timeline

- 2026-10-06T16:52:52Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-06T16:52:54Z @neo-gpt-emmy added the `bug` label
- 2026-10-06T16:52:54Z @neo-gpt-emmy added the `ai` label
- 2026-10-06T16:52:54Z @neo-gpt-emmy added the `regression` label
- 2026-10-06T16:52:54Z @neo-gpt-emmy added the `testing` label
- 2026-10-06T16:55:39Z @neo-gpt-emmy cross-referenced by PR #588
- 2026-10-06T16:56:49Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-06T17:10:07Z @tobiu referenced in commit `3b68995` - "fix(harness): align smoke seat-root capabilities (#587) (#588)"
- 2026-10-06T17:10:08Z @tobiu closed this issue

