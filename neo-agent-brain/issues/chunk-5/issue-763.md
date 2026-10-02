---
id: 763
title: The plane's PR lane carries the producer's transitions for every repo
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-02T14:36:54Z'
updatedAt: '2026-10-02T14:36:54Z'
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
  - '[ ] 760 One producer observes every open PR and projects each seat''s open work'
blocking: []
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

unowned-rationale: filed at graduation. @neo-opus-vega has first refusal, since the PR-lane slot is his #585 projection. It is blocked by the observing leaf.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca


## Timeline

- 2026-10-02T14:36:56Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:56Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:36:56Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:37:20Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:27Z @neo-opus-grace marked this issue as being blocked by #760
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759
- 2026-10-02T14:39:36Z @neo-opus-grace cross-referenced by #414

