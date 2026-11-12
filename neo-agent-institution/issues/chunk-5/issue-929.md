---
id: 929
title: The product bar's Nightshift toggle tells the fleet the operator is away
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T02:31:03Z'
updatedAt: '2026-11-12T05:29:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/929'
author: neo-opus-ada
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
# The product bar's Nightshift toggle tells the fleet the operator is away

## Context

Operator, 2026-11-12: "a new button with an icon (and maybe text) left to the switch theme button: Nightshift Mode on or off. so a toggle button. a state change should trigger a non-wake broadcast to tell peers if their operator is there or not." The mode's meaning is set in `neomjs/neo-agent-brain#1625`.
- **Both modes:** peers drive.
- **What changes:** whether operator-only asks (org admin, credentials, merges, questions peers cannot answer) get a quick answer.

Roadmap disposition: none of FM v1's five journeys needs this. It is team tooling the operator requested and prioritized in session on 2026-11-12, so it stays outside milestone FM v1.

## The Problem

The operator announces the mode to seats one at a time, and the FM has no control for it. Seats that boot or recover later never learn it.

## The Architectural Reality

Design authority: the operator's words above, and the header IA (`apps/agentos/design/institution-header-detail-ia.html`). The IA's product bar row holds "logo · 'Agent OS' title · instance switcher · setup line ·(spacer)· theme switch", the operator's own controls. The cockpit bar holds fleet state and actions only.

- **Source of truth:** the mode lives on the plane (`neomjs/neo-agent-brain#1625`). The button reads it back, and is not local state.
- **Who writes:** only the operator identity, the FM's verified viewer.
- **The wire:** the app reaches the fleet only through `FLEET_WIRE_METHODS` (`apps/agentos/fleet/installFleetBridge.mjs`). The verb `setOperatorPresence`, its once-per-change broadcast and the roster DTO's `operatorPresence` are `neomjs/neo-agent-brain#1627`.

## The Fix

1. A toggle in the product bar, left of the theme switch: a cloud-moon icon (the theme switch beside it already shows the plain moon) plus `Nightshift`, pressed when `away`. Its title carries the state, `since`, and `until` when set. When the DTO's `operatorPresence` is `null` (the presence read did not answer), it shows unknown, never present.
2. Turning it on offers spans: until switched off, 8 hours, 12 hours. A span's end is the `until` the plane enforces; the plane's expiry (#1625) returns it to `present` once `until` passes.
3. A click calls `setOperatorPresence` (`neomjs/neo-agent-brain#1627`), which writes the plane and sends the one broadcast. The button shows the record the verb answers, and the roster DTO between clicks.
4. The header IA's product bar row records the control.

## Acceptance Criteria

- [ ] The toggle shows the plane's mode on load and after a change from elsewhere, and unknown when the roster's `operatorPresence` is `null`.
- [ ] A click calls `setOperatorPresence` once with the mode (and `until`), and the button shows the record it answers.
- [ ] Specs cover the state mapping (present, away, away with `until`, expired, unknown). A Neural Link e2e flips it through a loopback Fleet.

## Post-Merge Validation

- [ ] On the installed FM (the batched build after the plane cut that carries `neomjs/neo-agent-brain#1626`), a click flips the plane's record and one `[operator-presence]` notice reaches `AGENT:*`.

## Out of Scope

- The plane record and its expiry (`neomjs/neo-agent-brain#1625`).
- The fleet verb and its broadcast (`neomjs/neo-agent-brain#1627`).
- The session rotation after a sunset (#842), which consumes the mode for its prompt.
- #95's wake policy.

## Related

`neomjs/neo-agent-brain#1627` (blocks this), `neomjs/neo-agent-brain#1625`, #842, `neomjs/neo-agent-brain#95`, and the header IA (#533 / #542).

Live latest-open + closed sweep: Institution 60 open (red band; latest `#928`..`#921`) plus an org-wide open search for `operator presence` at 2026-11-12T02:28Z; no toggle ticket.
A2A in-flight claim sweep: latest 10 messages at 02:28Z; no claim.
MC sweep: as in #1625; no prior decision.
Own-assignment sweep: 3 open (#917, #516, #424), none on this surface.

Owner: Ada (assigned 2026-11-12). Design review by the design seat.

Origin Session ID: c56bdafb-bd1d-4a67-b91d-3c37931e0abb
Retrieval Hint: "nightshift toggle product bar operator away present broadcast, left of theme switch"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T02:31:05Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T02:31:05Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T02:31:05Z @neo-opus-ada added the `ai` label
- 2026-11-12T02:31:05Z @neo-opus-ada added the `design` label
- 2026-11-12T02:31:10Z @neo-opus-ada cross-referenced by #842
- 2026-11-12T02:32:18Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T03:28:34Z @neo-opus-grace cross-referenced by #930
- 2026-11-12T04:25:27Z @neo-opus-ada cross-referenced by PR #1632
- 2026-11-12T05:34:00Z @neo-opus-ada cross-referenced by #932
- 2026-11-12T05:37:27Z @neo-opus-ada cross-referenced by PR #933
- 2026-11-12T06:15:23Z @neo-opus-ada referenced in commit `8fcf58b` - "fix(agentos): fence the Nightshift write, retire it on a switch (#929)

The Brain pin moves to 82423cf5, the dev head that carries the
setOperatorPresence verb, so `npm ci` installs the wire the toggle
writes through. Each write takes a generation; a later click or a
switch to another target makes an earlier answer land nowhere, and the
switch clears the presence and the toggle's last refusal."
- 2026-11-12T06:29:30Z @neo-gemini-julia cross-referenced by PR #934

