---
id: 449
title: 'The roster card shows a seat''s open work, and an awaiting-merge chip'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees: []
createdAt: '2026-10-02T14:36:56Z'
updatedAt: '2026-10-02T14:36:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/449'
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
# The roster card shows a seat's open work, and an awaiting-merge chip

## Context

This is the cockpit leaf of neomjs/neo-agent-brain#759, graduated from neomjs/neo#19122 (OQ4's card reader, and OQ5's awaiting-merge chip). The roster card shows a seat's open work, and the operator's queue shows what is ready for the human merge, both read from the Brain producer's projection.

## The Problem

The cockpit shows no seat's open PRs, red heads or due reviews. The operator learns that a PR is ready to merge from A2A handoff broadcasts or a harness-local poller.

## The Architectural Reality

- The roster card (`apps/agentos/view/fleet/roster/card/Container.mjs`) has a per-card reveal pane. Its records come from the fleet roster Store.
- The fleet server's snapshot carries `fleetOpenWorkSource` (Brain, the observing leaf) under the `ok · stale · unavailable` envelope.
- Target binding (#181, D#18965): retained truth belongs to the profile that answered.

## The Fix

1. **The card.**
   - One state line on each roster card: a seat's open PR count with the worst state (red, changes requested, review due).
   - The detail goes in the card's reveal pane: each PR with its CI and review state, and the seat's requested reviews.
   - Stale or unavailable reads as such, never as "no open work".
2. **An "awaiting merge" chip** where the queues live, from the producer's "approved + green + mergeable" rows (OQ5).
3. **Freshness and binding.** Both inherit target binding and the freshness envelope. No new view, and the Body re-derives nothing.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Card state line | `fleetOpenWorkSource` via the fleet snapshot | count + worst state per seat | stale/unavailable shown as such | JSDoc | unit |
| Reveal section | the same | each PR's CI and review state, and requested reviews | the same | JSDoc | unit + visual |
| Awaiting-merge chip | the producer's merge-ready rows | count + list | stale/unavailable | JSDoc | unit + visual |

## Acceptance Criteria

- [ ] AC-1: The card line and reveal section render a seat's open work from real Store records. Stale and unavailable render as such (unit).
- [ ] AC-2: The awaiting-merge chip lists the producer's merge-ready PRs (unit).
- [ ] AC-3: An instance switch retires the previous profile's open work (target binding; unit).
- [ ] AC-4: Darwin visuals and the input stamp agree for the changed surfaces.

## Out of Scope

- The Brain producer, its wakes, the digest and the plane lane.

## Decision Record impact

None.

## Related

neomjs/neo-agent-brain#759 (parent) · neomjs/neo#19122 · #414 (row 4) · #181.

Sweeps: Institution latest-20 at 2026-10-02T14:01Z and the epic's sweeps at 14:35Z. No equivalent.

unowned-rationale: filed at graduation for any cockpit seat to claim. It is blocked by the observing leaf and a pin carrying it.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca


## Timeline

- 2026-10-02T14:36:57Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:57Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:36:58Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:37:22Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:29Z @neo-opus-grace marked this issue as being blocked by #760
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759
- 2026-10-02T14:39:36Z @neo-opus-grace cross-referenced by #414

