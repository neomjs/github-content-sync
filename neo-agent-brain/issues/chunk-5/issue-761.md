---
id: 761
title: A PR's next-action holder is woken once per holder change
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-02T14:36:49Z'
updatedAt: '2026-10-02T14:36:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/761'
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
# A PR's next-action holder is woken once per holder change

## Context

This is the wake leaf of #759, graduated from neomjs/neo#19122 (body 2026-10-02T14:18:17Z). It turns the observing leaf's transitions into wakes for the seat that holds a PR's next action. It starts once the observing leaf's first observe-only day is posted. Design anchors: OQ2(2), OQ3, OQ5, OQ6 (R1, R2, sender), and OQ1 (the reviewer seat, unowned).

## The Problem

A transition reaches no one unless someone polls:
- a red head reaches no author;
- a review that has become due reaches no reviewer;
- a merge-ready PR reaches the operator only through a harness-local poller.

#427 wakes outside contributors' maintainers. It is one audience's version of this leaf.

## The Architectural Reality

- `readWakeDelivery()` (#512, PR #510) is the only projection that decides whether a wake was delivered.
- `planeMailboxClient.addMessage` never replays `add_message` on a retry, so the dedupe key belongs in the producer.
- `MailboxService` `getWakeSuppressionRisk` refuses to suppress a direct task message, so a producer wake that carries a `task` wakes by construction.
- The receiver records live in the host's wake state directory, which is why the producer is host-edge (the observing leaf).

## The Fix

1. **Wake eligibility** follows neomjs/neo#19122 OQ2's holder table:

   | state on the current head | holder (wake) |
   |---|---|
   | CI red, or `CHANGES_REQUESTED` | author |
   | CI green while a review request is open | each requested reviewer |
   | approved + green + mergeable | `@tobiu` |
   | merged | author |
   | an outside contributor's PR | the maintainer rotation |

   A wake is owed only on a holder change, and dedupes on (repo, PR, head SHA, holder, holder episode).
2. **The reviewer** is the currently requested native seat on the current head. An A2A review-request task names it only while that seat is still requested.
3. **Unowned work.** An org PR without a resolvable `Authored by` line wakes the lead with "unowned: <PR>".
4. **Delivery (R1).**
   - A holder whose route reads `unreachable` is skipped, and the lead is woken with the PR and the dead route.
   - A delivered wake that draws no artifact from the holder within T wakes the lead with the PR and the silent holder.
5. **One automatic waker (R2).** The producer is the only automatic waker for these transitions. Its wakes go out as the host fleet server's verified viewer, with the producer role as a `task` block.
6. **Switch-on gate.** Wakes switch on only when the observe-only day's complete per-pulse cost and per-seat transition count are inside the bounds this leaf states.
7. **#427.** This leaf absorbs #427's audience. @neo-opus-ada, who owns #427, reshapes or closes it against this leaf.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Wake eligibility | neomjs/neo#19122 OQ2(2) | one wake per holder change, to the table's holder | no holder change → no wake | JSDoc | unit (four controls) |
| Wake dedupe | the producer | key (repo, PR, head, holder, holder episode) | an identical poll → no wake | JSDoc | unit |
| Delivery check | `readWakeDelivery()` | delivered or escalated to the lead | `unreachable` → lead wake naming the route | JSDoc | unit |
| Sender | host fleet server viewer | a `task` block names the producer role | none | JSDoc | unit |
| Switch-on gate | the observing leaf's day | wakes stay off outside the bounds | off | JSDoc + #759 comment | unit + post-merge |

## Acceptance Criteria

- [ ] AC-1: Holder table (unit): each row wakes its holder once; a state with no holder change sends nothing.
- [ ] AC-2: Controls (unit):
  - a verdict changing under a red head → no wake;
  - red → green → red on one head → two author wakes;
  - identical polls → none;
  - two requested reviewers → one wake each, once.
- [ ] AC-3: The reviewer is the currently requested native seat. A stale task naming a removed seat wakes no one through it (unit).
- [ ] AC-4: An unowned org PR wakes the lead with "unowned: <PR>" (unit).
- [ ] AC-5: R1 (unit):
  - an `unreachable` route skips the holder and wakes the lead;
  - a delivered wake with no holder artifact within T wakes the lead.
- [ ] AC-6: R2: the producer's wakes carry the producer-role `task` and dedupe in the producer; no transition sends two automatic wakes (unit).
- [ ] AC-7: Switch-on: wakes stay off until the gate holds (unit). The falsifiers are recorded on #759 after a week (post-merge):
  - lead escalations outnumbering holder wakes;
  - two automatic wakes for one (PR, holder episode).
- [ ] AC-8: Every snapshotted repo runs the PR-body anchor check. Brain, Institution and devindex run the shared baseline, and `neo` runs `pr-baseline.yml`; `neo-agent-skills` is verified or filed.

## Out of Scope

- The observing producer, the readers, and Option D's sender half (already enforced).

## Decision Record impact

None.

## Related

#759 (parent) · neomjs/neo#19122 · #427 (absorbed; @neo-opus-ada reshapes it) · #512 / PR #510 · #327 (non-overlap: the defect ledger).

Sweeps: as on the observing leaf (2026-10-02T14:35Z), no equivalent; #427 is the absorbed audience.

unowned-rationale: filed at graduation, and it starts after the observing leaf's first day. @neo-opus-ada has first refusal, since she owns #427.

Origin Session ID: 31c9ca1a-ded8-4b19-8d99-682d259efeca
Retrieval Hint: `query_raw_memories("holder change wake producer readWakeDelivery lead escalation awaiting merge")`


## Timeline

- 2026-10-02T14:36:51Z @neo-opus-grace added the `enhancement` label
- 2026-10-02T14:36:51Z @neo-opus-grace added the `ai` label
- 2026-10-02T14:36:51Z @neo-opus-grace added the `agent-os` label
- 2026-10-02T14:37:17Z @neo-opus-grace added parent issue #759
- 2026-10-02T14:37:25Z @neo-opus-grace marked this issue as being blocked by #760
- 2026-10-02T14:37:48Z @neo-opus-grace cross-referenced by #759
- 2026-10-02T15:02:56Z @neo-opus-ada cross-referenced by #427
- 2026-10-02T15:09:44Z @neo-opus-grace cross-referenced by PR #764

