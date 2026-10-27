---
id: 1066
title: Held work rows retain original review requests and approvals
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-16T14:32:39Z'
updatedAt: '2026-10-16T15:29:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1066'
author: neo-gpt-euclid
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
closedAt: '2026-10-16T15:29:14Z'
---
# Held work rows retain original review requests and approvals

## Context

Held open-work rows expose the state a holder acts on, but their observed transitions are not the original review-request and approval occurrences. A consumer cannot recover the original requester, requested reviewer, provider occurrence ID, submitted time or reviewed commit from an observation pulse.

The existing query already reads standing opinions separately from latest engagement. Preserve that distinction: a later comment does not withdraw an approval, and an observation time must never replace the provider's original time.

## Architectural Reality

At Brain `c9d894c6`, `openWorkQueries.mjs` → `normalizePullRequest` → `reduceOpenWork` → `fleetOpenWorkSource.summaryOf` is the existing acquisition/held-row/projection path. The reducer already compares only explicit current-work fields when emitting transitions. Extend that path; no additional history reader, service, store or wake writer.

Read-only provider controls: [neo PR19581](https://github.com/neomjs/neo/pull/19581) supplies an original request and standing approval with matching reviewer Node ID. [neo PR19639](https://github.com/neomjs/neo/pull/19639) has a dismissed review absent from standing opinions, while a later comment remains in latest engagement. These are case controls, not a complete-history guarantee.

## Intended Change

Add optional `reviewRecords` to held open-work rows. It carries bounded original request occurrences plus approvals derived from the **currently returned standing opinions**, associated by provider reviewer Node ID. Preserve provider IDs/times, requester/recipient, reviewed commit and `onHead`; a request's historical head is unknown when the provider does not supply it.

The packet's history coverage is independent of current-work completeness. A truncated history must not make an otherwise complete work row partial, change its holder, suppress its wake, or invent an activity transition.

### Contract Ledger

| Surface | Authority / writer | Behavior | Fallback and boundaries |
|---|---|---|---|
| Open-work GraphQL snapshot | Existing producer acquisition | PR Node ID; standing-opinion ID, submitted time and stable author identity; last 20 review-request/removal/dismissal timeline items and both pagination directions. Existing current-request and engagement lists stay distinct. | Missing or malformed original metadata is unknown. History coverage comes from this timeline, never current request-list coverage. |
| `reviewRecords` | Pure normalizer, then the existing row reducer | Source scope includes provider API identity, repository, numeric PR number and provider PR Node ID. Original request entries carry ID/time/requester/recipient. Approval entries carry ID/time/reviewer/commit/`onHead`, an optional matched original request and explicit pairing status. Only fresh standing APPROVED opinions supply approval entries. | Legacy rows/queries may have no packet. Non-approval, explicit dismissal or complete standing-opinion absence cannot inherit cached approval eligibility. Partial absence remains unknown. |
| Request/approval association | Same source and provider reviewer Node ID | Latest preceding request to that reviewer, with the intervening request/removal interval covered; no cross-reviewer or team-to-member matching. A removal before approval defeats that pair; re-requests do not attach a new request to an older approval. | Complete exhausted history can establish absent; incomplete/missing history is unknown. Approval and request/removal timestamps tied to the same second are ambiguous because approval position is not in the filtered timeline. |
| Retention | Existing previous/current-row join | Current timeline window plus at most one proven request witness per currently returned approval, keyed by exact source/reviewer/review ID/time/commit. Retain a proved pair when a later truncated window drops it. | No unbounded union log; no union of old approval eligibility. Eviction, changed source/PR/opinion or a conflicting event retires the witness. A retained pair certifies that association only; gaps keep current history partial. |
| Held-row projection and persistence | Existing producer snapshot/store and `summaryOf` | Preserve the packet and existing row age through save/reload and held-row reads. | Unread/stale/unavailable behavior remains owned by the existing freshness contract. Packet-only changes produce zero current-work transitions or wakes. |
| Query-cost declaration | Existing static query hash/formula plus measured quota gate | Fifth per-PR connection changes the published 50-PR formula estimate from 2 to 3. | A formula or one-PR read is not a measured maximum; keep measured evidence keyed to the new query hash. |

## Acceptance Criteria

- [ ] The existing production acquisition supplies the packet with provider/PR scope; normalization and held projection preserve the original IDs, times and participants without restamping observation transitions.
- [ ] Table-driven controls distinguish same reviewer, another reviewer, team/member, re-request after CHANGES_REQUESTED, removal, head movement, dismissal, and same-second ambiguity. Latest comments preserve standing approval.
- [ ] Exhausted absence, truncated/missing/malformed history and partial opinion coverage remain distinct. Complete current-work lists with partial history preserve the same holder, aggregate coverage and wake eligibility.
- [ ] A complete pair followed by a shifted truncated window preserves only the exact current-approval witness; a disjoint-window gap stays partial. Replaced opinions, changed provider/repository/PR and retention bounds cannot revive stale associations.
- [ ] Producer save/reload and held projection retain the packet; legacy saved rows remain readable. Packet/coverage-only updates emit zero transitions, with a real current-work change as the positive control.
- [ ] Query-cost/hash expectations are updated and tested without promoting estimated cost to measured evidence. Focused owning/adjacent tests and current-head CI pass.

## Decision Record / Scope Impact

Decision Record: NOT REQUIRED — additive source metadata on the existing producer/normalizer/row projection, with no new write authority, service, storage owner or change to current-work routing. Structural fast-path: a pure helper beside `openWorkReducer.mjs`, with specs beside its existing Fleet unit tests. The query cost, retained evidence and optional packet contract are covered above.

## Out of Scope / Avoided Traps

- No merge-history reader, review-body ingestion or general event log.
- No new rendering, consumer replay adapter, installed acceptance or runtime deployment.
- No identity-root/roster rekey or login-based identity inference.
- No cached approval union, observation-to-original conversion, synthetic request head, completeness across an unseen gap, or history-driven current-work transitions.
- No changes to ownership, freshness, quotas or existing request-list membership semantics.

Related: #759 (existing producer/current-work outcome; this leaf adds original metadata only).

Origin Session ID: 5333f9c4-02c5-43fb-b40c-75216654a278

Retrieval Hint: original review requests standing approvals held open-work row; bounded request witnesses partial timeline history.



## Timeline

- 2026-10-16T14:32:41Z @neo-gpt-euclid added the `enhancement` label
- 2026-10-16T14:32:42Z @neo-gpt-euclid added the `ai` label
- 2026-10-16T14:32:42Z @neo-gpt-euclid added the `agent-os` label
- 2026-10-16T14:32:42Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-16T14:43:49Z @neo-gpt-euclid cross-referenced by PR #1067
- 2026-10-16T15:05:23Z @neo-gpt-euclid referenced in commit `8328526` - "feat(fleet): bind review records to their numbered PR (#1066)"
- 2026-10-16T15:29:14Z @tobiu referenced in commit `d05dea7` - "feat(fleet): retain original review records on held work (#1066) (#1067)

* feat(fleet): retain original review records on held work (#1066)

* feat(fleet): bind review records to their numbered PR (#1066)"
- 2026-10-16T15:29:14Z @tobiu closed this issue

