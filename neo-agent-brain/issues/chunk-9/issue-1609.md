---
id: 1609
title: An open-work wake reaches its seat with the wake off
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-11T19:05:31Z'
updatedAt: '2026-11-12T14:00:21Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1609'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-11-11T19:52:45Z'
---
# An open-work wake reaches its seat with the wake off

## Context

Leaf of #759 (own-work transitions reach the seat that holds the next action). The operator, 2026-11-11: CI results are the prime case for a wake. A seat ends its turn waiting on CI. The harness's PR monitor wakes it only on a failure, so a green run never reaches an idle seat. The open-work producer exists to close that loop: `repair-ci` on red, `route-review` on green, then `review`, `address-changes`, `rebase` and `post-merge`.

## The Problem

Every open-work message the producer sends arrives with the wake off. Three on 2026-11-11, all to `@neo-opus-grace`:

- `approve-runs` for neomjs/neo-agent-skills#338, at 18:19:34Z;
- `repair-ci` for neomjs/neo#20791, at 18:47:45Z;
- `route-review` for the same pull request, at 19:04:34Z.

Each reads `wakeSuppressed: true`, `priority: 'high'`, `from: '@tobiu'` and `senderPrincipalClass: 'human'`, and FM labels each "wake off". The red pull request sat until the operator noticed it in FM, and the green one waited for the seat's next turn.

## The Architectural Reality

Read at `7219f4e1`, the plane's revision:

- The round's quiet mode decides whether a wake is **sent**. `planOpenWorkWakes({quiet})` records a baseline and admits no send (`ai/services/fleet/openWorkWakes.mjs:100`; the startup line at `devFleetServer.mjs:758`). The three messages above were admitted and sent.
- `wakeMessageOf` (`wireFleetOpenWorkWakes.mjs:100-113`) sets no `wakeSuppressed`. The entrypoint sends through `planeClient.addMessage` (`devFleetServer.mjs:745`), so the plane stamps the viewer as the sender.
- `MailboxService#addMessage` resolves a `'human'` sender to operator steering (`ai/services/memory-core/MailboxService.mjs:3273`). For that, an omitted `wakeSuppressed` defaults to quiet (`:3313`). `getWakeSuppressionRisk` exempts human senders (`:823`), so nothing refuses it.
- A sender may elect a wake: `getWakeSuppressionRisk` returns `null` for `wakeSuppressed: false`.

## The Fix

`wakeMessageOf` sets `wakeSuppressed: false`. Everything this producer sends is a wake, whatever sender class the plane stamps.

## Acceptance Criteria

- [ ] `wakeMessageOf` returns `wakeSuppressed: false` for every kind, pinned in `test/playwright/unit/ai/services/fleet/wireFleetOpenWorkWakes.spec.mjs`.
- [ ] *(post-merge, installed)* After the next FM update, a `repair-ci` or `route-review` wake shows its wake on in FM, and the seat it names is woken.

## Out of Scope

- Who the sender is, and how a seat's drain orders these messages: #1607 and its (A)/(B) fork. Under either option, the producer's explicit election stays correct.
- The round's send gate, quiet until measured admission holds.
- The operator's `merge` wake: a human recipient has no wake route, so the flag changes nothing there.

## Avoided Traps

- **Waiting for #1607.** Its fork changes the sender and needs a design decision first. Meanwhile, the missing wake costs every seat a turn per CI result.
- **Changing the mailbox's default for human senders.** Quiet is the operator's own steering mode, by design. The automation should not inherit it, and a fix in the producer leaves that mode alone.

## Related

#759 (parent) · #1607 · #1535 and #1536 (the wakes switched on, and named the failing checks) · neomjs/neo#20791

Decision Record impact: none.

Live latest-open + closed sweep: the latest 20 of 150 open (red band), plus a closed search (`open-work wake wakeSuppressed`; #68, #148 and #14 read), at 2026-11-11T19:04Z. #1607 is the nearest open ticket and owns the sender question; this leaf owns the delivery flag. A2A in-flight claim sweep: the latest 8 messages, all read states, at 19:04Z; no claim on this surface. MC sweep: this lane's own trail only. Own-assignment sweep: #1518 and #1506, no overlap. Structure map: the change stays in the owning folder, `ai/services/fleet`.

handoff: @neo-opus-grace self-assigns; I wrote #1536 on this producer.

Origin Session ID: 2d4f1520-5217-4756-9013-970e443dcb03

Retrieval Hint: "open-work wake delivered with wake off: wakeMessageOf sets no wakeSuppressed, a human sender defaults to quiet"

## Timeline

- 2026-11-11T19:05:33Z @neo-opus-grace added the `bug` label
- 2026-11-11T19:05:33Z @neo-opus-grace added the `ai` label
- 2026-11-11T19:05:33Z @neo-opus-grace added the `agent-os` label
- 2026-11-11T19:05:36Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-11T19:05:40Z @neo-opus-grace added parent issue #759
- 2026-11-11T19:13:59Z @neo-opus-grace cross-referenced by PR #1610
- 2026-11-11T19:18:53Z @neo-opus-grace cross-referenced by #759
- 2026-11-11T19:20:40Z @neo-opus-grace cross-referenced by #1607
- 2026-11-11T19:52:45Z @tobiu referenced in commit `75c0b31` - "fix(fleet): an open-work wake asks to wake its seat (#1609) (#1610)

The plane stamps the viewer as the sender of every open-work message, and for a human sender an omitted wakeSuppressed defaults to quiet. So repair-ci, route-review and the other kinds reached their seat mailbox-only. wakeMessageOf now sets wakeSuppressed: false."
- 2026-11-11T19:52:46Z @tobiu closed this issue
- 2026-11-11T20:31:23Z @neo-opus-grace cross-referenced by PR #1613
- 2026-11-12T04:11:30Z @neo-gemini-julia cross-referenced by #1630
### @neo-opus-grace - 2026-11-12T14:00:21Z

**AC-2 receipt (after the 13:5xZ plane cut and FM install at Brain `cfb998af`):** the woken half is met; the FM display half was not observed by me.

- `MESSAGE:80d3b46a`, the `route-review` for #1644 sent at 13:57:05Z, is stored with `wakeSuppressed: false`.
- It woke the seat it names: my working session received it as a `[WAKE]` event.
- I cannot see the FM pane from this seat, so its "wake on" display still needs one look from a seat or person with the FM open.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 5b297761-7d20-44dc-b6ca-eedbdd1a5493

- 2026-11-12T16:21:51Z @neo-opus-grace cross-referenced by #1651

