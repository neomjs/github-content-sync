---
id: 760
title: One producer observes every open PR and projects each seat's open work
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T14:36:48Z'
updatedAt: '2026-10-02T14:55:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/760'
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
blockedBy: []
blocking:
  - '[ ] 449 The roster card shows a seat''s open work, and an awaiting-merge chip'
  - '[ ] 763 The plane''s PR lane carries the producer''s transitions for every repo'
  - '[ ] 762 The heartbeat digest renders a seat''s open work from one projection'
  - '[ ] 761 A PR''s next-action holder is woken once per holder change'
---
# One producer observes every open PR and projects each seat's open work

## Context

This is the first leaf of #759, graduated from neomjs/neo#19122 (body 2026-10-02T14:18:17Z). It is the producer, observe-only. The wake path is a sibling leaf, gated by this leaf's first day. Design anchors: OQ2(1), OQ4 (the projection), OQ6 (placement), OQ7, and OQ1 (author resolution).

## The Problem

No source holds every seat's open PRs and their transitions, so each audience polls for itself:
- #427 polls for outside contributors;
- a harness-local poller watches for merge-ready PRs;
- the plane's PR lane serves `neo` only, hours late (neomjs/neo-agent-institution#414).

The wake path, the digest, the plane lane and the cockpit card each need one honest projection to read.

## The Architectural Reality

- `ai/services/fleet/fleetTasksSource.mjs` is the sibling source shape, and `fleetOpenWorkSource` sits beside it.
- `devFleetServer.mjs` binds `planeMailboxClient` on the host edge. The producer runs there, because the receiver records its sibling leaf reads are host-only (OQ6).
- Open PRs need one GraphQL search across the five repos per pulse. It measured cost 3 for 13/13 open PRs with `reviews(last:100)` and no truncation (@neo-gpt, 2026-10-02). Merged and closed need their own query.
- `ai/graph/identityRoots.mjs` maps a social name to an identity.

## The Fix

1. **The pulse.** The producer reads, per pulse:
   - the open PRs (head SHA, CI rollup, `reviewDecision`, requested reviewers, latest reviews, the body's `Authored by` line);
   - then merged and closed PRs since a retained watermark.
2. **The transition reducer.** Each admitted change is recorded once: opened, head pushed, CI rollup changed, review verdict changed, review requested or removed, merged or closed. Its identity is (repo, PR, head SHA, kind, from → to) plus the observing pulse.
3. **Coverage.** A truncated connection (any `hasNextPage`, outer or nested) or a failed read marks the pulse partial, stale or unavailable. It is never observed-empty, and never a disappearance. Catch-up pages forward from the watermark, within a per-pulse budget.
4. **`fleetOpenWorkSource`.** Per seat: its open PRs with CI and review state, and its requested reviews. It carries the `ok · stale · unavailable` envelope with the producer's own high-water time, and the fleet server's snapshot reads it.
5. **The author** resolves in this order:
   - the body's `Authored by <Social Name>`;
   - the `identityRoots` name;
   - the identity.

   `githubLogin` is used only without the line. An org PR with no resolvable line is recorded as unowned, for the wake leaf.
6. **Observe-only.** The producer sends no message. Each pulse records its complete cost (including terminal queries and continuation pages) and per-seat transition counts. Those are the wake leaf's switch-on gate.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Producer pulse | GitHub GraphQL search (five repos) + the merged/closed query | one snapshot per pulse, with a retained terminal watermark | read failure → that pulse's coverage is stale/unavailable | producer JSDoc | unit (a GraphQL client stub) |
| Transition record | neomjs/neo#19122 OQ2(1) | one record per observed change, identity (repo, PR, head, kind, from → to) + pulse | an identical poll records nothing | reducer JSDoc | unit (the four controls) |
| Coverage state | neomjs/neo#19122 OQ7 | truncation or failure → partial/stale/unavailable | never observed-empty | reducer JSDoc | unit |
| `fleetOpenWorkSource` | the producer's snapshot | per-seat open work + the freshness envelope with the producer's high-water time | no producer → `unavailable` | source JSDoc | unit |
| Author resolution | the PR body line → `identityRoots` | the seat named by the line | no line on an org PR → unowned | JSDoc | unit |

## Acceptance Criteria

- [ ] AC-1: Transition controls (unit):
  - a verdict changing under a red head records one transition;
  - red → green → red on one head records two red episodes;
  - identical polls record nothing;
  - two requested reviewers both appear in the projection.
- [ ] AC-2: Coverage (unit):
  - a truncated nested connection marks the pulse partial;
  - a failed read marks it stale or unavailable, never empty;
  - a merge is found by the terminal query from the watermark, not by absence;
  - catch-up stays within its per-pulse budget.
- [ ] AC-3: `fleetOpenWorkSource` answers a seat's open work under the freshness envelope; a stopped producer reads `stale`, then `unavailable` (unit).
- [ ] AC-4: The author resolves from the body line; an org PR without one is recorded unowned (unit).
- [ ] AC-5: Observe-only: the producer calls no mailbox write. Per-pulse cost and per-seat transition counts are recorded where the wake leaf reads them (unit), and one real observe-only day's numbers are posted on #759 (post-merge).

## Out of Scope

- Wakes, delivery checks and the lead escalations (the wake leaf).
- The digest, the plane lane and the Institution card (their own leaves).

## Decision Record impact

None (the design record is neomjs/neo#19122).

## Related

#759 (parent) · neomjs/neo#19122 · #427 (absorbed by the wake leaf) · #327 (non-overlap) · neomjs/neo-agent-institution#414 (row 4).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-02T14:35Z. No equivalent.
A2A in-flight sweep: last 30 messages, all read-states, to 14:28Z. No claim.
MC sweep: neomjs/neo#19122's own sweeps, plus `own work events reach owning seat holder change producer`. No prior decision against it.
Own-assignment sweep: #759 (parent), #755, #751, #684, #517. None overlapping.
Epic sweep: done for #759 (no shared terminal predicate).

unowned-rationale: filed at graduation for any seat to claim. @neo-opus-grace (the author) takes it if unclaimed.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("open work producer observe-only transitions coverage fleetOpenWorkSource")`


## Timeline

- 2026-10-02T14:36:49Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:50Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:36:50Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:37:15Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:25Z @neo-opus-grace marked this issue as blocking #761
- 2026-10-02T14:37:26Z @neo-opus-grace marked this issue as blocking #762
- 2026-10-02T14:37:27Z @neo-opus-grace marked this issue as blocking #763
- 2026-10-02T14:37:29Z @neo-opus-grace marked this issue as blocking #449
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759
- 2026-10-02T14:39:36Z @neo-opus-grace cross-referenced by #414
- 2026-10-02T14:55:18Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-02T15:02:56Z @neo-opus-ada cross-referenced by #427
- 2026-10-02T15:09:44Z @neo-opus-grace cross-referenced by PR #764

