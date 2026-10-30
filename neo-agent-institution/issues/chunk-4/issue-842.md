---
id: 842
title: 'The Fleet Manager rotates one seat''s session on request, no other harness touched'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees: []
createdAt: '2026-10-30T21:34:28Z'
updatedAt: '2026-10-30T21:35:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/842'
author: neo-opus-vega
commentsCount: 1
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
---
# The Fleet Manager rotates one seat's session on request, no other harness touched

## Context

Operator, 2026-10-30 21:2xZ: *"you can always choose when to start a fresh session. exception are night shifts when i am not online => for that we need new tools for agent os and FM to enable a 'sunset and fresh session' combination."*

By day the Fleet Manager already does the second half: the operator boots a seat's fresh session from it after the seat wrote its handover. At night nobody clicks. The wake plane will ask for the rotation (neomjs/neo-agent-brain: *at the context gate the wake plane rotates the seat*, filed beside this ticket, under neomjs/neo-agent-brain#1373); the FM needs a way to answer that is not a seat Stop/Start.

## The Problem

The FM has no request surface for "start a fresh session for this seat and let the old one end", only the operator's click and the whole-seat Stop/Start, which closes the harness. A night rotation through Stop/Start is the move #732 rules out for updates, and it discards the harness's own state for nothing: the seat's app is fine, only its session is full.

## The Architectural Reality

- The fleet server (`harness/brain.mjs` lifecycle, the authenticated app↔fleet transport on the fleet port) already boots a seat's harness session the operator asks for; the path exists, the caller is the FM's UI only.
- #682 (restart every Claude seat for a staged Claude update and restore their sessions) needs the same per-seat primitive at bulk; #732 (an FM update never requires closing a harness) sets the constraint; neomjs/neo-agent-brain#571 (one seat folder layout Fleet provisions and launches into) and neomjs/neo-agent-brain#1230 (Claude Desktop restores a seat's old folder at launch) are the launch-side facts.
- The operator's standing rule stays: one live session per seat. A rotation is sequential: the old session has written its handover and ends; the fresh one starts; at no point do two sessions of one seat work in parallel.
- Design authority: a mechanism, not a designed surface; the FM's Agent Detail may later show "rotated at …", which is the design seat's read (#505/#507), not this ticket's.

## The Fix

1. A fleet-server endpoint `rotate-session` for one seat, callable by the plane with the seat's sunset receipt (the handover message id) and refused without it; it starts a fresh harness session for that seat through the existing boot path, in the running harness app, touching no other seat and no other harness.
2. The old session is not killed: it ends itself after its handover (the seat's sunset), and the FM marks it retired once the fresh session's wake route is registered; if the old session is still live after a bounded window, the FM reports it and does nothing destructive.
3. The journal and the roster card carry the rotation (`rotated 21:17Z · from <old> to <new>`), so a morning read of the FM shows every night rotation without the plane's log.

## Acceptance Criteria

- [ ] A unit arm: `rotate-session` without a sunset receipt is refused; with one, exactly one fresh session starts for that seat and no other seat's process changes.
- [ ] A unit arm: the old session is retired only after the fresh session's route registration; a live old session past the window is reported, never killed.
- [ ] An installed receipt: one rotation of a Claude seat on the local plane, roster card and journal showing it, the other seats' processes unchanged (pid list before/after).
- [ ] BEFORE/AFTER screenshot pair of the roster card in the PR body.

## Out of Scope

Deciding when to rotate (the Brain half); the seat's own sunset and its receipt (neomjs/neo-agent-skills#247); bulk restarts for a staged update (#682); the design of a rotation row in Agent Detail.

## Related

#682, #732, #733, #734, neomjs/neo-agent-brain#571, neomjs/neo-agent-brain#1230, neomjs/neo-agent-brain#1373, neomjs/neo-agent-skills#247.

Ownership: `unowned-rationale:` the seat-lifecycle mechanics belong with #682's owner; a plane steward or the FM steward takes it with a line here.

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-30T21:32:40Z; no equivalent (#682 restarts seats for an update; nothing rotates one session). A2A sweep: last 30 `lane-claim` messages, none on seat sessions. MC sweep: "seat sunsets itself at night without the operator and a fresh session boots from the Fleet Manager automatically context gate handover", 6 results, every one a human-booted fresh session, no prior mechanism. Own-assignment sweep: 4 open (#732, #651, #649, #485); #732 is the constraint, not this primitive.

Origin Session ID: 40918982-6f4d-45f6-8b01-c5f951c22edf
Retrieval Hint: "fleet manager rotate one seat session fresh session old ends after handover no harness closed"

## Timeline

- 2026-10-30T21:34:29Z @neo-opus-vega added the `enhancement` label
- 2026-10-30T21:34:30Z @neo-opus-vega added the `agent-os` label
- 2026-10-30T21:34:30Z @neo-opus-vega added the `ai` label
### @neo-opus-vega - 2026-10-30T21:35:10Z

The plane half is neomjs/neo-agent-brain#1383 (at the context gate the wake plane rotates the seat: sunset receipt in, fresh session out), sub of neomjs/neo-agent-brain#1373.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-10-30T21:35:10Z @neo-opus-vega cross-referenced by #1383
- 2026-10-30T21:35:13Z @neo-opus-vega cross-referenced by #247

