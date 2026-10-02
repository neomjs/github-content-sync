---
id: 763
title: The plane's PR lane carries the producer's transitions for every repo
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T14:36:54Z'
updatedAt: '2026-10-02T17:50:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/763'
author: neo-opus-grace
commentsCount: 0
parentIssue: 759
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 760 One producer observes every open PR and projects each seat''s open work'
blocking: []
closedAt: '2026-10-02T17:50:17Z'
---
# The plane's PR lane carries the producer's transitions for every repo

## Context

This is a reader leaf of #759, graduated from neomjs/neo#19122 (OQ4's third reader, folded at body 2026-10-02T14:18:17Z). The plane's PR lane takes its PR events from the producer's transitions. That gives FM v1 row 4's `pr` chip (neomjs/neo-agent-institution#414) a live source for every repo.

## The Problem

The plane's PR lane serves `neo` only and lags the corpus publish by hours. Measured 2026-10-02: 200 events, all `neo`, the newest 3 h old. The slot reads the orchestrator's single-origin materialized root (#585). The adapter is not limited to one repo; its PR contributor is.

## The Architectural Reality

- `fleetPrLaneActivityAdapter` (`:77–83` at `a9dd22f`) merges three contributors, PR events, issue/comment events and graph-stall events, then slices by `limit`.
- `planePrLaneActivityReader` calls `get_pr_lane_activity` with `{limit}` and no history cursor.
- `fleetActivityComposer` stamps the composite with its own read time, so producer freshness must come from the slot.
- The chain is `planePrLaneActivityReader` → `wireFleetActivityReadSource` → `fleetActivityComposer`. A failed slot degrades the composite.

## The Fix

1. The PR contributor reads the producer's transitions (opened, review verdict, merged or closed) as `pr-activity` events, for all five repos. The issue/comment and stall contributors stay on their current sources, and the slot keeps merging all three.
2. The slot's capability carries the producer's high-water time, and each event its observation time.
3. **A declared retained window.** The producer keeps a bounded window of transitions, and a reader that falls behind it reads a coverage gap, never a silent loss.
4. The slot keeps its `{capability, counts, events}` boundary and the existing chain. No reader sources GitHub.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| PR contributor | the producer's transitions | `pr-activity` events for all five repos | no producer → the slot's capability degraded | JSDoc | unit |
| Other contributors | their current sources | unchanged | unchanged | — | existing arms |
| Slot freshness | the producer's high-water time | capability and events carry the producer's times | — | JSDoc | unit |
| Retained window | the producer | bounded; behind it → coverage gap | never silent loss | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: PR events come from the producer's transitions for every repo the producer snapshots (unit).
- [ ] AC-2: The issue/comment and stall contributors are unchanged (existing arms pass).
- [ ] AC-3: Freshness is the producer's: a cached buffer read later still reports the producer's high-water time, not the read time (unit).
- [ ] AC-4: A reader behind the retained window reads a coverage gap (unit).
- [ ] AC-5: Row 4's chip shows a PR transition on a non-`neo` repo within the producer's cadence (post-merge, on neomjs/neo-agent-institution#414).

## Out of Scope

- The multi-origin materialized root (parked, @neo-opus-vega's first refusal); it is not needed for this lane.
- The producer itself, the wakes, the digest and the card.

## Decision Record impact

None.

## Related

#759 (parent) · neomjs/neo#19122 OQ4 · #585 · neomjs/neo-agent-institution#414.

Sweeps: as on the observing leaf (2026-10-02T14:35Z). No equivalent.

unowned-rationale: filed at graduation. @neo-opus-vega has first refusal, since the PR-lane slot is her #585 projection. It is blocked by the observing leaf.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca



## Timeline

- 2026-10-02T14:36:56Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:56Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:36:56Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:37:20Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:27Z @neo-opus-grace marked this issue as being blocked by #760
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759
- 2026-10-02T14:39:36Z @neo-opus-grace cross-referenced by #414
- 2026-10-02T15:52:17Z @neo-gpt cross-referenced by PR #764
- 2026-10-02T16:42:53Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T16:52:58Z @neo-opus-vega cross-referenced by PR #769
- 2026-10-02T17:09:23Z @neo-gpt cross-referenced by #762
- 2026-10-02T17:33:08Z @neo-opus-vega referenced in commit `2a4b96a` - "fix(fleet): the PR lane's producer contract reaches the delivered snapshot, the base is asked without PR events, and a full window covers from its second pulse (#763)

The composite capability carries each contributor's own capability under slots, so the producer's high-water time, coverage, retained window and coverage gap survive composition instead of the read clock. The base reader is asked for no pull-request events (prEvents, declared on the plane tool and kept per shape by its memo store) at the composer's maximum bound, so a replaced corpus PR displaces no surviving issue, lane-claim or stall event. A full window may have cut inside one pulse, so coverage begins at the first pulse after the oldest retained one; a partial pulse degrades the slot. Euclid's three controls are arms."
- 2026-10-02T17:37:05Z @neo-opus-vega referenced in commit `faa7d1c` - "test(fleet): the adapter spec's section comment names its subject, not a review round (#763)

The source-comment archaeology guard reads a touched file whole: the pre-existing 'Cycle-2 RA3' marker in fleetPrLaneActivityAdapter.spec.mjs decayed into a violation the moment this lane appended an arm to the file. The comment now says what the section tests."
- 2026-10-02T17:50:17Z @tobiu referenced in commit `f9d3088` - "feat(fleet): the PR lane carries the open-work producer's transitions for every repository (#763) (#769)

* feat(fleet): the PR lane carries the open-work producer's transitions for every repository (#763)

The PR/lane slot's pull-request contributor reads the producer's retained transitions (opened, review verdict, merged, closed) as pr-activity events at the producer's observing pulse; the base reader keeps the issue, lane-claim and stall events. The slot's capability carries the producer's high-water time, coverage and declared retained window, and a reader behind that window reads a coverage gap. The producer is read where it lives, in both modes; nothing is published to the plane.

* fix(fleet): the PR lane's producer contract reaches the delivered snapshot, the base is asked without PR events, and a full window covers from its second pulse (#763)

The composite capability carries each contributor's own capability under slots, so the producer's high-water time, coverage, retained window and coverage gap survive composition instead of the read clock. The base reader is asked for no pull-request events (prEvents, declared on the plane tool and kept per shape by its memo store) at the composer's maximum bound, so a replaced corpus PR displaces no surviving issue, lane-claim or stall event. A full window may have cut inside one pulse, so coverage begins at the first pulse after the oldest retained one; a partial pulse degrades the slot. Euclid's three controls are arms.

* test(fleet): the adapter spec's section comment names its subject, not a review round (#763)

The source-comment archaeology guard reads a touched file whole: the pre-existing 'Cycle-2 RA3' marker in fleetPrLaneActivityAdapter.spec.mjs decayed into a violation the moment this lane appended an arm to the file. The comment now says what the section tests."
- 2026-10-02T17:50:17Z @tobiu closed this issue

