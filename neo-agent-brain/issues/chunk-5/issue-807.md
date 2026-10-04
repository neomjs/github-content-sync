---
id: 807
title: Open-work escalations reach a named recipient on a delivery the receiver confirmed
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-03T11:07:21Z'
updatedAt: '2026-10-04T11:03:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/807'
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
blocking: []
---
# Open-work escalations reach a named recipient on a delivery the receiver confirmed

## Context

The #804 review split this from #761: [review 5400413001](https://github.com/neomjs/neo-agent-brain/pull/804#pullrequestreview-5400413001) and [disposition 5968528311](https://github.com/neomjs/neo-agent-brain/pull/804#issuecomment-5968528311). #761 keeps the holder wakes. This leaf owns the escalation contract that neomjs/neo#19122 graduated as OQ1 and OQ6 (`[RESOLVED_TO_AC]`):
- an unowned org PR wakes the lead;
- a holder whose route reads `unreachable` is skipped, and the lead is woken with the PR and the dead route;
- a delivered wake that draws no artifact from the holder within T wakes the lead with the PR and the silent holder.

There, "a delivered wake is a dispatch the receiver recorded as `delivered`".

## The Problem

Two things the contract assumes don't exist yet.

**No recipient.** Nothing in code names "the lead". The only trace is `lead-role-baton` in `MailboxService`'s wake-suppression allowlist, and the operator has no wake route.

**No per-message delivery witness.** `readWakeDelivery()` returns verdicts per subscription. `who_is_online.undeliverable` is their projection per identity (`projectIdentityWakeReachability`). Neither can say that a particular wake was delivered.

A silent-holder escalation built on a planned or attempted send would certify silence after a wake that may never have arrived. The #804 review's control showed exactly that: a rejected send still escalated four hours later.

## The Architectural Reality

- `ai/services/memory-core/wakeDeliveryReader.mjs` (`readWakeDelivery`) and `wakeDeliveryProjection.mjs` (`projectIdentityWakeReachability`): the delivery authority, per subscription and per identity.
- #761's round (`wireFleetOpenWorkWakes`) persists its ledger before it sends and never retries an ambiguous send. Its dead-route skip reads the identity projection.
- The fleet server is host-edge, so it can read the receiver's dispatch records.

## The Fix

Decide first, then build:
1. **The recipient.** This blocks everything else. Who receives an escalation: a declared seat, the lead-role baton holder, or a surfaced consumer such as the cockpit beside the awaiting-merge list? Decided as an OQ1/OQ6 fold on neomjs/neo#19122, then recorded here.
2. **A receipt contract.** A sent wake is correlated with the receiver's dispatch record, so "delivered" holds per message. An attempt, or an ambiguous send, never counts as delivered.
3. **The holder's artifact.** What counts as acting within T (a new head, a review on the head, a reply), read from the producer's transitions.
4. **The three escalations,** built on #761's round. The salvage map is #761 comment 5968481942.

## Acceptance Criteria

- [ ] AC-1 (was #761 AC-4): an unowned org PR reaches the decided recipient with "unowned: <PR>" (unit).
- [ ] AC-2 (was #761 AC-5): an `unreachable` route reaches the recipient with the PR and the dead route. A wake the receiver recorded as `delivered`, with no holder artifact within T, reaches the recipient with the PR and the silent holder. A failed or undelivered wake never does. The tests discriminate failed, undelivered and delivered (unit).
- [ ] AC-3 (was #761 AC-7's escalation falsifier, post-merge): a week after switch-on, #759 records whether escalations outnumber holder wakes.

## Out of Scope

The holder wakes themselves (#761) and retrying ambiguous sends.

## Decision Record impact

Depends on the neomjs/neo#19122 OQ1/OQ6 fold for the recipient; no ADR.

## Related

Parent #759 · split from #761 · neomjs/neo#19122 OQ1 and OQ6 · #512 / PR #510 (the delivery reader).

Blocked on the recipient decision. Its named owner is this ticket's assignee, who wrote neomjs/neo#19122; the decision is folded there.

Live latest-open sweep: the latest 20 open Brain issues, read at 2026-10-03T11:06:38Z, plus searches for "escalation recipient" and "lead wake": no equivalent beyond #761, which this splits from. A2A: the #804 reviewer's disposition (MESSAGE:513fffaa). Own-assignment: #759 and #761, the parent and the source.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "open-work escalation recipient lead wake receiver confirmed delivery silent holder"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-03T11:07:21Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-03T11:07:23Z @neo-opus-grace added the `enhancement` label
- 2026-10-03T11:07:23Z @neo-opus-grace added the `ai` label
- 2026-10-03T11:07:23Z @neo-opus-grace added the `agent-os` label
- 2026-10-03T11:08:54Z @neo-opus-grace cross-referenced by #761
- 2026-10-03T11:12:28Z @neo-opus-grace cross-referenced by PR #808
- 2026-10-03T11:12:41Z @neo-opus-grace cross-referenced by PR #804
- 2026-10-04T11:03:05Z @neo-opus-grace cross-referenced by #15000
- 2026-10-04T11:03:16Z @neo-opus-grace unassigned from @neo-opus-grace

