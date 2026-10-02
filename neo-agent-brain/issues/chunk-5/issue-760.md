---
id: 760
title: One producer observes every open PR and projects each seat's open work
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-02T14:36:48Z'
updatedAt: '2026-10-02T16:39:57Z'
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
  - '[x] 763 The plane''s PR lane carries the producer''s transitions for every repo'
  - '[ ] 762 The wake digest renders a seat''s own open work from one plane copy'
  - '[ ] 761 A PR''s next-action holder is woken once per holder change'
closedAt: '2026-10-02T16:39:57Z'
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

1. **The pulse.** One pulse runs at a time; a pulse asked for while one runs is that pulse. The scope is every GitHub repository the registry's seats work on (`metadata.repo` + `metadata.repos`). Per pulse the producer reads:
   - the open PRs, with head SHA, CI rollup, `reviewDecision`, requested reviewers, each reviewer's latest review and whether it judged the current head, and the body's `Authored by` line;
   - then the PRs closed in a fixed window of close times (`closed:<since>..<until>`), resumed at its cursor across pulses and restarts. Each window reaches back an overlap past the previous one's end, because GitHub's search index can lag a close.
2. **The transition reducer.** Each admitted change is recorded once: opened, head pushed, CI rollup changed, review verdict changed, review requested or removed, merged or closed. Its identity is (repo, PR, head SHA, kind, from → to) plus the observing pulse. A close is one per close time, and a reopening retires its marker. A new PR reports the reviews already requested on it; a row first seen after a read that missed pages is no `opened`.
3. **Coverage.**
   - An answer without its page structure is a failed read.
   - A truncated or missing nested list, or one holding an item that names no reviewer, is unknown beyond what it shows: requests add to the last complete list and remove nothing.
   - A cut-off page or a failed read marks the pulse partial, stale or unavailable. It is never observed-empty, and never a disappearance.
   - The first pulse starts the watermark. It moves only once the window completes, to the overlap before the window's end. A failed pulse records what its answered requests cost and marks the rest unknown.
   - A PR that leaves complete reads with no terminal row is recorded as `vanished` on its pulse, never as a close: `closed:` ranges miss some PRs closed without merging.
4. **`fleetOpenWorkSource`.** Per seat: its open PRs with CI, verdict and latest reviews, and its requested reviews. It carries the `ok · stale · unavailable` envelope with the producer's own times, and each row its own observation time: a carried row is `stale` past the stale bound and leaves the projection, counted, past the unavailable bound. The bridge serves it as the read verb `fleetOpenWork` (`read-observe`, `awaiting-s3`).
5. **The author** resolves in this order:
   - the body's `Authored by <Social Name>`, through `identityRoots`;
   - otherwise the login, through the registry's seats, then `identityRoots`.

   An org PR with no resolvable line is recorded as unowned, never as its login, for the wake leaf.
6. **Observe-only.** The producer sends no message. Each pulse records its complete cost (including terminal queries and continuation pages) and per-seat transition counts. Those are the wake leaf's switch-on gate.
7. **Persistence.** The state lives in one file beside the registry. An absent file is a first pulse. A file that exists but cannot be read leaves the producer `unavailable`, is never saved over, and is read again on the next pulse.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Producer pulse | GitHub GraphQL search over the registry's GitHub repositories + the close-time window | one snapshot per pulse, pulses never overlapping; the window resumes at its cursor | read failure → that pulse's coverage is stale/unavailable, with its answered cost | producer JSDoc | unit (a GraphQL client stub) |
| Transition record | neomjs/neo#19122 OQ2(1) | one record per observed change, identity (repo, PR, head, kind, from → to) + pulse; one close per close time | an identical poll records nothing | reducer JSDoc | unit (the four controls, close → reopen → close) |
| Coverage state | neomjs/neo#19122 OQ7 | malformed answer, truncation or failure → failed/partial/stale/unavailable | never observed-empty; a partial request list removes no one; an unexplained absence is `vanished` | reducer + producer JSDoc | unit |
| Latest reviews | GitHub `latestReviews` | each reviewer's latest state and whether it judged the current head, carried on the row | a truncated list makes the pulse partial | reducer JSDoc | unit |
| `fleetOpenWorkSource` | the producer's snapshot | per-seat open work + the freshness envelope; each row aged by its own observation | no producer → `unavailable`; a row past the bound → `unobserved` | source JSDoc | unit |
| `fleetOpenWork` (read verb) | `FLEET_WIRE_METHODS` + both admission ledgers | `read-observe`, `awaiting-s3`, like `fleetTasks` | unwired → `unavailable` | bridge JSDoc | unit (allowlist + policy specs) |
| Persisted state | `<registry dataDir>/open-work.json` | carries the snapshot, window and record across a restart | an unreadable file → `unavailable`, never overwritten | wiring JSDoc | unit |
| Author resolution | the PR body line → `identityRoots`; a login → the registry, then `identityRoots` | the seat named by the line | no line on an org PR → unowned | JSDoc | unit |

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

## Deltas after filing

- **Scope.** The Brain serves other tenants, so the scope is the registry's GitHub repositories, not a fixed list of five.
- **Round 1 of PR #764 (@neo-gpt).** It added serialized pulses, structure validation, the reviewer baseline across partial lists, the resumable close-time window, the answered cost of a failed pulse, per-row age, the unreadable-state contract and close episodes. The latest reviews this ticket promised are now carried on the row.
- **After Round 1 (found by the author).** It added the window overlap for index lag, the `vanished` record, the initial watermark on a partial read, the reviews requested on a new PR, and no `opened` after a missed page. On neomjs/neo, `closed:` ranges reach all 6201 merged PRs, but only 252 of the 323 PRs closed without merging.

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
- 2026-10-02T16:05:41Z @neo-opus-grace referenced in commit `31a41cc` - "fix(fleet): the open-work producer observes faithfully across overlap, partial reads, catch-up and restart (#760)

Pulses never overlap; a malformed answer is a failed read; a partial request list adds to the last complete one and removes no one; the close-time window resumes at its cursor across pulses and restart, and a failed pulse keeps its answered cost; every row carries its own observation time; unreadable saved state is never overwritten; a close is one per close time; the latest reviews ride on the row."
- 2026-10-02T16:19:39Z @neo-opus-grace referenced in commit `cffe9fb` - "fix(fleet): the open-work producer reads a close the search indexed late, and records a PR that vanishes (#760)

Each terminal window now reaches back an overlap past the previous one's end, so a merge GitHub's search indexes after its window ended is still read, once. The first pulse starts the watermark even when its open read was partial. A PR that leaves complete reads with no terminal row is recorded on its pulse as vanished, since closed: date ranges miss some PRs closed without merging. A row first seen after a read that missed pages is no opened transition, and the reviews already requested on a new row are reported."
- 2026-10-02T16:21:51Z @neo-opus-grace referenced in commit `11e78fc` - "test(fleet): an hour of partial pulses never makes a missed row fresh, through the producer (#760)"
- 2026-10-02T16:25:56Z @neo-opus-grace referenced in commit `da3bf6a` - "fix(fleet): a request list holding a reviewer it cannot name removes no one (#760)

A complete reviewRequests list whose item names no reviewer (requestedReviewer null, or a type the query cannot name) filtered to fewer reviewers and recorded false removals. A list is now complete only when every item resolves, so such a row is partial and its requests keep the last complete baseline; a latest review without an author makes the row partial too. The query names Bot and Mannequin reviewers by login."
- 2026-10-02T16:39:57Z @tobiu referenced in commit `1b5810d` - "feat(fleet): one producer observes every open PR and projects each seat's open work (#760) (#764)

* feat(fleet): one producer observes every open PR and projects each seat's open work (#760)

Observe-only: each pulse reads the open PRs of the repositories the registry's seats work on and the ones merged or closed since its watermark, reduces them into per-observation transitions, and keeps a bounded window with each pulse's cost and per-seat counts. A partial read holds the watermark; a failed read is stale or unavailable. The bridge's new read verb fleetOpenWork projects each seat's open work under the producer's own freshness.

* fix(fleet): the open-work producer observes faithfully across overlap, partial reads, catch-up and restart (#760)

Pulses never overlap; a malformed answer is a failed read; a partial request list adds to the last complete one and removes no one; the close-time window resumes at its cursor across pulses and restart, and a failed pulse keeps its answered cost; every row carries its own observation time; unreadable saved state is never overwritten; a close is one per close time; the latest reviews ride on the row.

* fix(fleet): the open-work producer reads a close the search indexed late, and records a PR that vanishes (#760)

Each terminal window now reaches back an overlap past the previous one's end, so a merge GitHub's search indexes after its window ended is still read, once. The first pulse starts the watermark even when its open read was partial. A PR that leaves complete reads with no terminal row is recorded on its pulse as vanished, since closed: date ranges miss some PRs closed without merging. A row first seen after a read that missed pages is no opened transition, and the reviews already requested on a new row are reported.

* test(fleet): an hour of partial pulses never makes a missed row fresh, through the producer (#760)

* fix(fleet): a request list holding a reviewer it cannot name removes no one (#760)

A complete reviewRequests list whose item names no reviewer (requestedReviewer null, or a type the query cannot name) filtered to fewer reviewers and recorded false removals. A list is now complete only when every item resolves, so such a row is partial and its requests keep the last complete baseline; a latest review without an author makes the row partial too. The query names Bot and Mannequin reviewers by login."
- 2026-10-02T16:39:58Z @tobiu closed this issue
- 2026-10-02T17:09:23Z @neo-gpt cross-referenced by #762
- 2026-10-02T17:20:38Z @neo-gpt cross-referenced by PR #769
- 2026-10-02T19:05:28Z @neo-opus-ada cross-referenced by #779

