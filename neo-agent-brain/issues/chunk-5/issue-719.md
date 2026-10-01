---
id: 719
title: 'The self-succession lease arms race a 40 ms real-time window, so CI load fails them'
state: CLOSED
labels: []
assignees:
  - neo-opus-vega
createdAt: '2026-10-01T18:42:58Z'
updatedAt: '2026-10-01T19:19:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/719'
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
closedAt: '2026-10-01T19:19:51Z'
---
# The self-succession lease arms race a 40 ms real-time window, so CI load fails them

## Context

Clio's defect-note (A2A, 2026-10-01 18:00Z) names a second arm that fails on PR head suites while base suites pass: `test/playwright/unit/ai/daemons/orchestrator/daemon.spec.mjs:38`, "a DEAD predecessor is waited out". The differential gate (#650) then blocks each PR that hits it. The note's other arm, in the receiver spec, is #717.

## The Problem

Both self-succession arms use `ttlMs: 40` against the real clock.
- **The dead-predecessor arm** asserts that the wrapper *waited* (`sleptMs > 0`). If 40 ms or more pass between the predecessor's acquire and the first claim, for example during a GC pause on a loaded runner, the lease is already stale. The first claim then succeeds with no wait, and the arm fails.
- **The live-holder control** fails the same way: a stale lease at the first claim is reclaimed instead of refused.

The lease core already takes a clock: `acquireAuthorityLease({now})`, and `fileLease.mjs` is "fully injectable". But `acquireAuthorityLeaseSurvivingSelfSuccession` (`ai/daemons/orchestrator/daemon.mjs`) does not pass a clock through. Its `remaining` uses `Date.now()`, so the specs cannot avoid real time.

## The Fix

`acquireAuthorityLeaseSurvivingSelfSuccession` takes `now = Date.now`, uses it for `remaining`, and passes it to both claims. Both arms drive a fake clock: the sleep stub advances it, and the live holder pulses at the advanced time. Nothing in the arms waits on real time.

## Acceptance Criteria

- [ ] The wrapper threads `now` to both `acquireAuthorityLease` calls and to its `remaining` computation. The production default stays `Date.now`.
- [ ] Both arms run on a fake clock with no real-time sleep. Red-first: on the current wrapper, the dead-predecessor arm fails because the second claim is still refused.
- [ ] The live-holder control still discriminates. With the pulse removed, it acquires instead of refusing.

## Out of Scope

- The lease core (`fileLease.mjs`, `authorityLease.mjs`), which already takes the clock.

## Related

#717 (the note's other arm) · #650 (the differential gate)

Live latest-open sweep at 2026-10-01T18:4xZ: the 15 most recent open Brain issues plus #717. No equivalent.

Origin Session ID: 6b4062a3-941e-4b08-b997-765875a5b207


## Timeline

- 2026-10-01T18:42:59Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-01T18:46:01Z @neo-opus-vega cross-referenced by PR #720
- 2026-10-01T19:19:51Z @tobiu referenced in commit `92122a0` - "fix(orchestrator): the self-succession wrapper takes the lease core's clock, so its arms no longer race real time (#719) (#720)

Both self-succession arms ran a 40 ms window against the real clock. A GC
pause on a loaded runner made the predecessor's lease stale before the first
claim, so "it must wait" failed and the live-holder control acquired. That
gave head-suite reds the differential gate counts as introduced.
acquireAuthorityLeaseSurvivingSelfSuccession now takes `now` (default
Date.now), uses it for the remaining wait, and passes it to both claims, as
the lease core already accepts. Both arms drive a fake clock that the sleep
stub advances. The touched file's legacy ticket-ref-ok escape becomes the
typed decision-record form the archaeology guard requires."
- 2026-10-01T19:19:52Z @tobiu closed this issue

