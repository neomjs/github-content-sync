---
id: 779
title: The open-work projection names each open PR's next-action holder
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T19:05:26Z'
updatedAt: '2026-10-02T20:29:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/779'
author: neo-opus-ada
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
closedAt: '2026-10-02T20:29:48Z'
---
# The open-work projection names each open PR's next-action holder

## Context

This leaf is a sub of #759. My [epic review](https://github.com/neomjs/neo-agent-brain/issues/759#issuecomment-5959421595) (Stage 3, finding 2) found it as a missing phase. Two other leaves read the same predicate from neomjs/neo#19122:
- OQ5's awaiting-merge chip, in neomjs/neo-agent-institution#449;
- OQ2's holder table, in #761: "approved + green + mergeable → `@tobiu`".

Today no leaf projects that predicate.

## The Problem

The open-work projection says what each seat owns and reviews. It never says who holds a PR's next action.
- **The chip has no source.** neomjs/neo-agent-institution#449 is unblocked: Institution `dev` pins Brain `8a078f0`. Its only path to an "awaiting merge" list is a cockpit filter over row fields. That rebuilds the per-audience projection #759 exists to retire, and #449 itself says "the Body re-derives nothing".
- **Unowned PRs are invisible.** The projection is keyed by seat, so a PR no seat owns appears nowhere once its review requests are consumed.

**The measurement.** A probe called the shipped `projectOpenWork` (Brain `dev@761dce8`) with three approved, green, `MERGEABLE` rows: one seat-owned, one `outside`, one `unowned`, none with requested reviewers. It placed only the seat-owned row. The projection's keys are `state, observedAt, coverage, reason, detail, unobserved, seats`: no holder, no merge-ready set.

## The Architectural Reality

- `projectOpenWork` (`ai/services/fleet/fleetOpenWorkSource.mjs:66–67` at `761dce8`) puts a row under `row.owner.seat`'s `authored` and under each requested `@` id's `reviewing`. Nothing else is projected.
- `ownerOf` (`ai/services/fleet/openWorkReducer.mjs:54`) gives `outside` and `unowned` owners `seat: null`.
- A row carries `ci` (`red` · `pending` · `green`, from `CI_STATES`), `verdict` (`reviewDecision`), `mergeable` (`MERGEABLE` · `CONFLICTING` · `UNKNOWN`), `draft`, `owner`, `requested`, and `reviews` with an `onHead` flag. These are the inputs OQ2's table reads.
- `snapshot.rows` holds open PRs only. Merged and closed rows come from the terminal read (`reduceOpenWork`), so OQ2's "merged → author" row is a transition for #761, not a projection state.
- The bridge serves the projection untouched through the `fleetOpenWork` read verb (`ai/services/fleet/FleetControlBridge.mjs:785`).
- Sibling placement: `ai/services/fleet/` holds `openWorkProducer.mjs` and `openWorkReducer.mjs` (`npm run ai:structure-map -- --files --loc`, 2026-10-02).

## The Fix

1. **`ai/services/fleet/openWorkHolder.mjs`** exports `holderOf(row) → {role, ids}`. It implements OQ2's table for an open row, first match wins, in this order:

   | the row | holder |
   |---|---|
   | an `outside` owner, `ci` red or green, nothing requested, no review with `onHead` | `rotation`, `ids: []` |
   | `ci: 'red'`, or a `CHANGES_REQUESTED` review with `onHead` | `author`: the owner's seat, or `[]` when the owner is no seat |
   | `ci: 'green'` and `requested` non-empty | `reviewer`: each requested id |
   | `ci: 'green'`, an `APPROVED` review with `onHead`, `mergeable: 'MERGEABLE'`, not `draft` | `operator`, `ids: []` |
   | anything else | `none` |

   - **Reviews of the head, not `verdict`.** OQ2's table reads "state on the current head". `verdict` (`reviewDecision`) keeps a change request the author has pushed past, and it keeps an earlier head's approval.
   - **Rotation first.** An outside PR nobody has engaged with holds no seat, so the rotation precedes `author`. Once a seat is requested, or has reviewed the head, the later rows decide.
   - **Draft rule.** "Mergeable" excludes a draft, because GitHub will not merge one.
   - **Standing opinions.** The change-request and approval rules read `row.opinions`, each reviewer's standing opinion from `latestOpinionatedReviews`, which the snapshot query now also reads. They do not read `row.reviews`, whose latest review can be a later comment. The rotation rule's engagement still reads `row.reviews`.
   - **Partial reads.** On a partial row (`row.partial`: a review or request list truncated or unresolved), only positive evidence decides: a red head, or a change request on the head, is the author's. Every other rule reads an absence a missing page could hold, so it answers `unknown`.
   - **Role, not identity.** `operator` and `rotation` stay roles. The reader that wakes or renders resolves the identity, so the Brain never names the operator's handle.
2. **`projectOpenWork`** attaches `holder` to every summary. It gains `awaitingMerge`: every open row whose holder is `operator`, whatever its owner kind.
   - Those rows age like seat rows: `stale` past the stale bound, excluded and counted once in `unobserved` past the unavailable bound.
   - A seat-narrowed read keeps `awaitingMerge`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `holderOf(row)` (new) | neomjs/neo#19122 OQ2's holder table + OQ7's coverage rule | first-match role + ids per the table | `{role: 'none', ids: []}`; on a partial row without positive evidence, `{role: 'unknown', ids: []}` | JSDoc | unit, one arm per table row + controls |
| `projectOpenWork` summaries (`fleetOpenWorkSource.mjs` `summaryOf`) | `holderOf` | each summary carries `holder` | — | JSDoc | unit |
| `projectOpenWork` result | `holderOf` over `snapshot.rows` | `awaitingMerge`: every `operator`-held row, any owner kind, aged like seat rows | unavailable projection → `[]` under its `unavailable` state | JSDoc | unit (the probe's three rows) |
| `fleetOpenWork` verb (`FleetControlBridge.mjs:785`) | the projection | unchanged pass-through | unchanged | — | existing arms |
| Snapshot row `opinions` (`OPEN_WORK_SNAPSHOT` + `normalizePullRequest`) | GitHub `latestOpinionatedReviews` | each reviewer's standing approval or change request, with `onHead` | a truncated or missing list marks the row `partial` | query + normalizer JSDoc | unit (the normalizer arm) |

## Acceptance Criteria

- [ ] AC-1: `holderOf` returns each table row's holder (unit, one arm per row). Controls:
  - red + `APPROVED` → `author`;
  - a change request the author pushed past, with the review requested again → `reviewer`;
  - an approval of an earlier head → not `operator`;
  - an approved, green, `MERGEABLE` draft → not `operator`;
  - `mergeable: 'UNKNOWN'` → not `operator`;
  - green with nothing requested and no approval → `none`;
  - on a partial row: a visible approval, an untouched outside PR, or a visible request → `unknown`, while a red head or an on-head change request → `author`.
- [ ] AC-2: `awaitingMerge` lists the outside and unowned rows the probe placed nowhere, beside the seat-owned one. Each summary carries `holder` (unit; red-first against `dev`, where `awaitingMerge` is absent).
- [ ] AC-3: An `awaitingMerge` row past the stale bound reads `stale`. Past the unavailable bound, it leaves `awaitingMerge` and counts once in `unobserved` (unit).
- [ ] AC-4: `operator` and `rotation` holders carry no identity (unit).

## Deltas after filing

- **Head-scoped reviews (2026-10-02, the author, while building).** As filed, the table read `verdict`. Two rows now read reviews of the head instead (the Fix table's first bullet gives the reason). A head-blind mutant fails exactly the two arms that pin this.
- **Partial reads (2026-10-02, @neo-gpt-sophie's pre-review finding on PR #780).** As filed, the table ignored the producer's `row.partial`, so OQ7's truncation could decide `operator` or `rotation` from an absence it never saw. The partial-read rule and the `unknown` role are added above.
- **Standing opinions (same review packet).** `latestReviews` keeps a reviewer's latest review, so a comment after an approval or change request on the same head hid it. Live on neo#19234 and neo#19324, the same reviewer and head read `COMMENTED` in `latestReviews` and `APPROVED` in `latestOpinionatedReviews`. The write surface therefore grows into the producer's snapshot query and normalizer (ledger row above). A `dryRun` puts the query's cost at 2 with or without the new connection.

## Out of Scope

- **Wakes**, dedupe and delivery: #761, which imports `holderOf` instead of defining it.
- **The cockpit** card and chip: neomjs/neo-agent-institution#449.
- **The plane copy**: #762.
- **"Fork runs awaiting approval."** OQ2's outside row names it, but the row carries no workflow-approval state, so it reads `none` here. #761 decides whether it needs a field.
- **An approved, green, `CONFLICTING` head.** OQ2's table has no row for it, so it stays `none`. A holder for it changes the graduated table and belongs on #759.

## Avoided Traps

- **A cockpit-side "awaiting merge" filter.** It re-derives the holder outside the producer, and over the seat-keyed projection it drops outside and unowned PRs.
- **The operator's handle in the Brain.** OQ2 writes `@tobiu`; the Brain serves other tenants, so the projection names the role.
- **Deriving merge-ready from `seats`.** That loses exactly the rows this leaf exists to show.

## Decision Record impact

None. This is aligned with neomjs/neo#19122 OQ2 and OQ5; no ADR is touched.

## Related

#759 (parent) · #761 (imports `holderOf`) · neomjs/neo-agent-institution#449 (reads `holder` and `awaitingMerge`) · #760 / PR #764 (the projection) · neomjs/neo#19122.

Sweeps:
- Live latest-open: the latest 20 open Brain issues at 2026-10-02T19:05Z. No equivalent. #761 owns the wake path; this leaf moves its holder function ahead of it (proposed on #761).
- A2A in-flight: the inbox from 17:17Z to 19:05Z, all read-states. No claim on the holder, #449 or #761.
- MC sweep: "operator merge queue awaiting merge outside contributor PR approved green nobody holds next action", 10 results. They are manual merge-queue verifications from June and July. No prior decision.
- Own-assignment sweep: 18 open Brain tickets on me. #52 and #30 touch adjacent surfaces (the relation, wake routing); neither covers the holder.
- Epic sweep: N/A, not an epic.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83
Retrieval Hint: `query_raw_memories("open work holder awaiting merge operator role projection outside unowned")`


## Timeline

- 2026-10-02T19:05:28Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T19:05:28Z @neo-opus-ada added the `ai` label
- 2026-10-02T19:05:28Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T19:05:28Z @neo-opus-ada added the `agent-os` label
- 2026-10-02T19:05:35Z @neo-opus-ada added parent issue #759
- 2026-10-02T19:05:39Z @neo-opus-ada cross-referenced by #761
- 2026-10-02T19:14:27Z @neo-opus-ada marked this issue as blocking #449
- 2026-10-02T19:14:41Z @neo-opus-ada cross-referenced by #449
- 2026-10-02T19:17:09Z @neo-opus-ada cross-referenced by PR #780
- 2026-10-02T19:49:57Z @neo-opus-ada referenced in commit `76944a7` - "feat(fleet): a partial read decides the holder only on positive evidence (#779)

A truncated or unresolved review or request list (the producer's row.partial, reachable from
latestReviews(first:20)) cannot prove an absence. A red head or a change request on the head still
names the author; every rule that reads an absence (rotation's untouched PR, the operator's clean
approval, the reviewer rule a hidden change request would precede) answers unknown, so a partial row
never enters awaitingMerge."
- 2026-10-02T19:58:30Z @neo-opus-ada referenced in commit `b133b47` - "feat(fleet): the holder reads each reviewer's standing opinion, not their latest review (#779)

latestReviews keeps a reviewer's latest review, a comment included, so a comment after an approval
or change request on the same head hid that opinion. The snapshot now also reads
latestOpinionatedReviews; normalizePullRequest carries it as row.opinions, and its truncation marks
the row partial. holderOf decides change requests and approvals from the opinions, keeps the latest
reviews as engagement for the rotation rule, and treats a row with no opinion list as incomplete.
The open-work stubs carry the new connection."
- 2026-10-02T20:19:25Z @neo-opus-ada referenced in commit `dc377a9` - "feat(fleet): a partial read lets an outside PR's red head reach the author (#779)

The rotation rule reads an absence (nothing requested, no review of the head), so only a complete read hands an untouched outside PR to the rotation. A partial read of a red outside PR now falls to the positive red-head rule, as the partial-read contract states, instead of answering unknown."
- 2026-10-02T20:21:32Z @neo-fable-clio cross-referenced by #782
- 2026-10-02T20:29:48Z @tobiu referenced in commit `804356b` - "feat(fleet): the open-work projection names each open PR's next-action holder (#779) (#780)

* feat(fleet): the open-work projection names each open PR's next-action holder (#779)

holderOf(row) answers neomjs/neo#19122 OQ2's holder table as roles (rotation, author, reviewer,
operator, none), first match wins, counting a review only when it judged the current head.
projectOpenWork attaches the holder to every summary and lists every operator-held row in
awaitingMerge, whoever owns it: the seat-keyed lists drop outside and unowned PRs.

neomjs/neo-agent-institution#449's awaiting-merge chip reads it; #761 imports holderOf.

* feat(fleet): a partial read decides the holder only on positive evidence (#779)

A truncated or unresolved review or request list (the producer's row.partial, reachable from
latestReviews(first:20)) cannot prove an absence. A red head or a change request on the head still
names the author; every rule that reads an absence (rotation's untouched PR, the operator's clean
approval, the reviewer rule a hidden change request would precede) answers unknown, so a partial row
never enters awaitingMerge.

* feat(fleet): the holder reads each reviewer's standing opinion, not their latest review (#779)

latestReviews keeps a reviewer's latest review, a comment included, so a comment after an approval
or change request on the same head hid that opinion. The snapshot now also reads
latestOpinionatedReviews; normalizePullRequest carries it as row.opinions, and its truncation marks
the row partial. holderOf decides change requests and approvals from the opinions, keeps the latest
reviews as engagement for the rotation rule, and treats a row with no opinion list as incomplete.
The open-work stubs carry the new connection.

* feat(fleet): a partial read lets an outside PR's red head reach the author (#779)

The rotation rule reads an absence (nothing requested, no review of the head), so only a complete read hands an untouched outside PR to the rotation. A partial read of a red outside PR now falls to the positive red-head rule, as the partial-read contract states, instead of answering unknown."
- 2026-10-02T20:29:49Z @tobiu closed this issue

