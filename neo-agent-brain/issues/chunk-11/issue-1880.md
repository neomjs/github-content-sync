---
id: 1880
title: 'A sunset receipt can say: no successor, or a successor at a set time'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-20T20:09:02Z'
updatedAt: '2026-11-20T20:09:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1880'
author: neo-opus-vega
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
---
# A sunset receipt can say: no successor, or a successor at a set time

Not a good first issue: wake-plane internals, owned work (#1383's mechanism), filed for after the Engine 14 cut on 2026-11-23.

## Context

The operator, 2026-11-20, relayed by Clio: "every claude sunset now starts a fresh session. there imho is no flag to say 'no fresh session' yet. sounds like a friction->gold item which you should delegate."

The specimen is Clio's seat on 2026-11-20:
- The handovers of 18:24Z and 19:03Z each opened a successor within two minutes.
- The second sunset existed only to keep the seat dark until Sat 21 Nov 09:57 CET.
- Its successor booted anyway, re-read the substrate and armed three session-only timers.
- A live seat below the context gate is then woken by every unread DM overnight.
- The operator stopped that seat by hand.

## The Problem

Since #1383, a sunset receipt (a self-addressed message tagged `sunset-protocol-handover`) rotates the seat at once. A seat that sunsets to stay dark, or to resume at a set time, cannot say so. Its dated acts then hang on session-only timers in a successor nobody needed yet.

## The Architectural Reality

- `ai/daemons/wake/sunsetReceipt.mjs#readSunsetReceipt` reads a seat's receipt for the rotation lane and the wake hold: its newest self-addressed handover since a stamp.
- `ai/daemons/orchestrator/scheduling/sessionRotation.mjs` emits the rotation; `ai/daemons/wake/sessionRotation.mjs` is the emitter/receiver contract.
- The `session-sunset` skill (neomjs/neo-agent-skills) writes the receipt.
- Nothing in the receipt names a successor policy. `parseFocusItemsReceipt` reads only the subject's focus count.

Design authority: `ai/daemons/wake/sessionRotation.mjs` — "A seat that writes its sunset handover continues in a fresh session". This ticket amends that contract: a handover may say that its seat does not continue now.

## The Fix

The operator reads the two disposition names and the dark seat's wake behaviour before the build (AC-4).

1. **One successor disposition in the receipt.** Without one, the seat rotates at once, as today.
   - `successor: none` — no rotation. The seat stays dark until an operator start.
   - `successor: at <ISO time>` — the rotation is emitted at that time, and the dated acts the handover names wait for it instead of session-only timers.
2. **The plane honours it.** `readSunsetReceipt` parses it, from a tagged concept or one fixed body line (a choice the build makes and records). The rotation emitter honours it, and the wake hold keeps the seat dark until then.
3. **The skill documents it.** `session-sunset` documents both dispositions (a Skills leaf).

## Acceptance Criteria

- [ ] AC-1: A receipt without a disposition rotates as today, pinned by a regression spec.
- [ ] AC-2: A receipt with `successor: none` emits no rotation, and no session opens for the seat until an operator start.
- [ ] AC-3: A receipt with `successor: at <time>` emits its rotation at that time and not before.
- [ ] AC-4: Before the build, the operator's answer on the two names and on what a DM does to a dark seat (queues unread, or opens a session) is on this ticket.
- [ ] AC-5: `session-sunset` documents both dispositions, in a linked Skills pull request.

## Out of Scope

- Codex and Antigravity rotation (#1871 and its siblings).
- The context gate.
- Night-shift wake policy (#95).

## Related

#1383 · #1682 · #1712 · #1722 · #1739 · #1777 · #1872 · #95

Live latest-open + closed sweep: checked the latest 20 open issues plus an open and closed search (sunset successor, "no successor", "fresh session", resumeHarness) on 2026-11-20. No equivalent open; the rotation history (#1383, #1682, #1712, #1722, #1739, #1777, #1872) is closed and #1871 is another symptom. Backlog band: 159 open (red); no epic owns rotation. A2A claim sweep: Clio's delegation names the author; no competing claim.

Origin Session ID: 0304bcd2-6bba-4b36-9450-bf66f2a11001

## Timeline

- 2026-11-20T20:09:03Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-20T20:09:05Z @neo-opus-vega added the `enhancement` label
- 2026-11-20T20:09:05Z @neo-opus-vega added the `ai` label
- 2026-11-20T20:09:05Z @neo-opus-vega added the `agent-os` label

