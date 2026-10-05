---
id: 879
title: Wake eligibility reads a seat's participation from its identity node
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T11:48:04Z'
updatedAt: '2026-10-05T17:33:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/879'
author: neo-opus-vega
commentsCount: 0
parentIssue: 875
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T17:33:02Z'
---
# Wake eligibility reads a seat's participation from its identity node

## Context

A leaf of epic #875, split from #874 at intake. The wake daemon decides who may be woken from `ai/daemons/wake/wakeTargetEligibility.mjs`, a map built from the static `IDENTITIES` import when the module loads. `daemon.mjs` consults it at L632, L2543 and L2585, and the retry admission goes through the same map. A bench recorded on the plane's identity node (#28) therefore cannot stop a wake, and an operator whose seats have no root has no gate at all.

## The Problem

The daemon already reads subscription nodes from the Memory Core, but takes participation from our team's file. Agent Detail's bench promises "nobody wakes it"; that holds only when this gate reads the node.

## The Fix

The daemon reads participation from the identity nodes in its own graph store, the records `who_is_online` reads, once per poll cycle. Every delivery boundary uses that read: queue admission, the coalesced flush, retry enqueue and retry attempt. A read that could not answer delivers nothing and drops nothing; queued and retried work waits for the next read. A seat with no node stays eligible, the open-set case.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `getAgentIdentityNodes(db)` (new; `ai/daemons/wake/queries.mjs`) | this ticket | The `AgentIdentity` rows, read through the store's `idx_nodes_label` | Throws when the store cannot answer. A malformed row cannot exist, because the index rejects it at write | JSDoc | `queries.spec` (query plan, write rejection, throw) |
| `wakeTargetPermission(identity, participation)` (new; replaces `isWakeTargetEligible`) | this ticket; Euclid's #890 review | `eligible`, `benched` (a non-active node status) or `unread` (no read answered) | An identity without a node is `eligible` | JSDoc | `wakeTargetEligibility.spec` |
| Poll cycle (`daemon.mjs pollLoop`) | this ticket | Reads participation first. A throw aborts the cycle before `lastSyncId` moves | Participation stays `unread` until the next read answers | Comment | `daemon.spec` (unread defers) |
| Queue admission (`evaluateSubscription`) | this ticket | Queues only for `eligible` | — | — | `daemon.spec` (node versus root) |
| Coalesced flush (`flushSubscription`) | Euclid RA-1 | `benched`: drops the queue. `unread`: keeps the queue and re-arms at the poll interval. `eligible`: delivers | Gated before the queue is consumed | Comment | `daemon.spec` (bench while queued; unread defers, then delivers once) |
| Retry enqueue and attempt (`enqueueDeliveryRetry`, `attemptDeliveryRetries`) | Euclid RA-1 | `benched`: drops the retry. `unread`: keeps it (an attempt is skipped, and its count is not spent) | — | Comment | Euclid's isolated falsifier, row 5 |
| Diagnostics | this ticket | A failed read: the poll loop's ERROR line, each cycle. A drop for a benched seat: an INFO line naming the identity and, for a flush, the count | A deferral adds no line beyond the poll loop's ERROR | — | `daemon.spec` (the drop line) |

## Acceptance Criteria

- [ ] A node bench stops a wake for a seat whose root reads `active`, and a root bench no longer stops one once the node reads `active` (specs).
- [ ] An unanswered participation read delivers nothing and drops nothing, and the poll loop's error line names why; a missing node stays eligible (two controls).
- [ ] `wakeTargetEligibility.mjs` no longer imports `identityRoots.mjs` (grep receipt).

## Out of Scope

The Fleet DTO's read (#874), the heartbeat and issue focus (their own leaf), the write (#28).

unowned-rationale: filed from #874's narrowing so the gate keeps a home; open for self-selection under #875.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1



## Timeline

- 2026-10-05T11:48:06Z @neo-opus-vega added the `enhancement` label
- 2026-10-05T11:48:06Z @neo-opus-vega added the `ai` label
- 2026-10-05T11:48:06Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T11:48:14Z @neo-opus-vega added parent issue #875
- 2026-10-05T12:18:16Z @neo-opus-vega cross-referenced by #883
- 2026-10-05T12:47:55Z @neo-opus-vega cross-referenced by #885
- 2026-10-05T14:03:43Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-05T14:22:29Z @neo-opus-vega cross-referenced by PR #890
- 2026-10-05T15:55:16Z @neo-opus-vega referenced in commit `be28c59` - "feat(wake): the coalesced flush and the retries read participation too, and an unread read defers instead of dropping (#879)

wakeTargetPermission answers eligible, benched or unread. The flush checks it before consuming its queue: a bench recorded while a wake waited drops it, and an unread read keeps the queue for the next attempt. Retry enqueue and attempt drop only for a bench. The identity read goes back to the store's indexed label predicate, which also rejects a malformed row at write."
- 2026-10-05T17:33:02Z @tobiu referenced in commit `f43142e` - "feat(wake): the wake daemon reads a seat's participation from its identity node, not the roots (#879) (#890)

* feat(wake): the wake daemon reads a seat's participation from its identity node, not the roots (#879)

The daemon reads the AgentIdentity rows from its graph store once per poll cycle, the rows who_is_online reads, and isWakeTargetEligible takes that map at all three sites. A read that throws aborts the cycle before the cursor moves. The query checks json_valid first, so one malformed row elsewhere cannot fail it. The roots census wakeSeatIdentities had no runtime consumer and is removed.

* feat(wake): the coalesced flush and the retries read participation too, and an unread read defers instead of dropping (#879)

wakeTargetPermission answers eligible, benched or unread. The flush checks it before consuming its queue: a bench recorded while a wake waited drops it, and an unread read keeps the queue for the next attempt. Retry enqueue and attempt drop only for a bench. The identity read goes back to the store's indexed label predicate, which also rejects a malformed row at write."
- 2026-10-05T17:33:02Z @tobiu closed this issue

