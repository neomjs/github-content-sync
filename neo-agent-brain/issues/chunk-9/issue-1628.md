---
id: 1628
title: A wake does not tell a working seat its own lane comes first
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-11-12T03:19:07Z'
updatedAt: '2026-11-12T14:00:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1628'
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
closedAt: '2026-11-12T05:01:59Z'
---
# A wake does not tell a working seat its own lane comes first

## Context

The operator, 2026-11-12, on #1613: "the wake is meant to ensure peers who idled out get it. is it clear that peers always should finish their current own lane first? because if peers would jump on it first, then hit compaction, they might lose context. even if not they lose their focus."

Read at `dev` `568d637c`:
- **The wake names the item, never the moment.** `composeOpenWorkWake` (`ai/services/fleet/openWorkWakeMessage.mjs`) says to re-check the live pull request before acting. Nothing says when. Its JSDoc classes the wake as "neither operator steering nor drain-first" (priority normal).
- **The seat sees the digest, not the body.** `[WAKE][priority:…] N events … (latest: "<subject>" from <sender>)` carries the subject only. The standing directive (`WAKE_LANE_DIRECTIVE`, `ai/daemons/wake/wakeLaneDirective.mjs`) rides on pure-heartbeat digests only: "message / task / permission wakes omit it".
- **Delivery holds nothing back from a working seat.** The context gate (`contextGatePolicy.mjs`) defers a wake only into a session above its size threshold. Its active-session skip applies to idle-out nudges only, so a content wake reaches a working seat at its next turn end.
- **The receiving rules are silent on the lane in progress.** They order the lifecycle queue before a new lane: AGENTS.md's wake line, and post-review-pickup §3.
- **Observed** in my own session on 2026-11-12: at turn ends, Stop-hook wake digests moved me straight onto the woken items while my own lane still had steps left. The items were a review pickup, a CI repair and a reviewer routing.

## The Problem

A seat can end a turn mid-lane: waiting on CI, a reviewer or the operator, or between two steps. A normal-priority wake then arrives, and the seat reads it as the next task. Taking it there costs the lane twice:
- A long side task can reach compaction and drop the lane's unsaved state.
- Without compaction, the seat still loses its thread.

The wake exists for seats that idled out. A working seat finds the same item in its inbox at its next lane boundary.

## The Architectural Reality

Design authority:
- `wakeDigestBuilder.mjs`'s summary: the digest "preserves the strongest interruption signal by choosing the highest message priority … while keeping normal/low wakes deferrable".
- `composeOpenWorkWake`'s JSDoc: "a system observation is neither operator steering nor drain-first".

Deferrable is the design, but no seat is told so.

- `ai/daemons/wake/wakeLaneDirective.mjs` is the single canonical source of the standing directive.
- Two renderers append it, to pure heartbeats only: `buildWakeDigest` (`wakeDigestBuilder.mjs:163`) and `formatLocalWakeDigest` (`localWakeAdapters.mjs:63`). Their comments require both to move together.

## The Fix

1. Beside `WAKE_LANE_DIRECTIVE`, add one canonical directive for content wakes whose digest priority is normal or low: "Finish the lane you are working on first; these wait in your inbox. At your next lane boundary, take them in lifecycle order."
2. Both renderers append it to a message, task or permission digest at normal or low priority. A high-priority digest (operator steering, drain-first) and a pure heartbeat keep today's text.
3. Harness-agnostic prose, like the standing directive.

## Acceptance Criteria

- [ ] A normal- or low-priority message or task digest ends with the new directive, from both renderers. Unit spec.
- [ ] A high-priority digest carries no defer directive, and a pure heartbeat keeps `WAKE_LANE_DIRECTIVE` alone. Same spec.
- [ ] *(post-merge)* After the next plane cut, an open-work wake's digest in a working seat shows the directive. One receipt on this ticket.

## Out of Scope

- Holding wakes back until a seat goes idle: presence-gated delivery, where #95 owns presence-aware wake policy.
- The skills' wording: post-review-pickup §3 says "before a new lane". Revisit only if seats still switch with the directive in front of them.
- What a high-priority wake means.

## Related

#759 (parent) · #1613 (the open-work wake) · #1619 (merge wakes ride the same digest) · #95

Sweeps, 2026-11-12 03:2xZ:
- **Live latest-open + closed:** the latest 20 open (155 open, red band; #1627..#1512). All-state search: `wake lane in progress OR finish current lane OR mid-lane wake`. Read #95 and #759. No equivalent; folds under #759.
- **A2A:** the 10 newest messages, all states. No claim on wake digests.
- **MC:** `a wake pulls a peer off the lane it is working on; it switches to the woken item, hits compaction…`, 6 results, no prior decision.
- **Own-assignment:** 7 open (#1619, #1614, #1518, #1506, #807, #684, #54). #1619 shares the digest, nothing overlaps.

Retrieval Hint: "wake digest normal priority deferrable finish the current lane first compaction"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-11-12T03:19:07Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-12T03:19:08Z @neo-opus-grace added the `enhancement` label
- 2026-11-12T03:19:09Z @neo-opus-grace added the `ai` label
- 2026-11-12T03:19:09Z @neo-opus-grace added the `agent-os` label
- 2026-11-12T03:19:14Z @neo-opus-grace added parent issue #759
- 2026-11-12T04:10:55Z @neo-fable-mnemosyne cross-referenced by #1629
- 2026-11-12T04:28:15Z @neo-opus-grace cross-referenced by PR #1633
- 2026-11-12T05:01:59Z @tobiu referenced in commit `82423cf` - "feat(wake): a normal or low wake tells a working seat its own lane comes first (#1628) (#1633)

A content wake reached a working seat at its next turn end and read as its next task: nothing in the digest said the event could wait. WAKE_DEFER_DIRECTIVE now ends every message, task or permission digest below high priority. A lane with a step to take comes first, a woken event that is that step is taken, and a lane that waits on someone else releases the seat. selectWakeDirective in wakeLaneDirective.mjs picks the directive for both renderers, buildWakeDigest and formatLocalWakeDigest, replacing the pure-heartbeat check each carried. High priority and pure heartbeats keep their text."
- 2026-11-12T05:02:00Z @tobiu closed this issue
### @neo-opus-grace - 2026-11-12T14:00:18Z

**AC-3 receipt (post-cut, plane `cfb998af`):** met.

An open-work wake reached my seat while I was working, and its digest ended with the directive. The wake was `MESSAGE:80d3b46a`, a `route-review` for #1644, sent at 13:57:05Z, normal priority. The digest's last line read: "Directive — own lane first: while your current lane has a step you can take now, take it before acting on these, unless one of them is that step; they wait in your inbox."

The directive matched what I did: routing #1644 was my lane's next step, so I took it first. The reviewer is requested and the task is completed.

🖖 Grace (Claude Opus 5.5, Claude Code) · session 5b297761-7d20-44dc-b6ca-eedbdd1a5493


