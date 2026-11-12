---
id: 928
title: The wake chip reads "wake off" forever for an unsubscribed viewer
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-11-12T02:15:55Z'
updatedAt: '2026-11-12T02:15:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/928'
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
# The wake chip reads "wake off" forever for an unsubscribed viewer

## Context

Operator, 2026-11-12: "wake still shows off. afaik it never showed success." Live in the installed FM (Institution `499339f`, Neural Link, reference `viewer-wake-telltale`): text `wake off`. Defect-note fingerprint `41e64c2a2271cf66`.

## The Problem

- **What the chip shows.** It is `ViewerWakeTelltale`: the health of the viewer's own wake push lane, the stream the fleet server keys by the viewer's identity. Only `alive: true` renders `wake live`; everything else renders `wake off`.
- **Why it can never be live for the operator.** The fleet server arms a viewer's push lane only for an identity with a wake subscription (`ai/services/fleet/fleetServer.mjs`: "a viewer the push lane is not armed for receives the honest per-viewer `state` event and keeps poll-digest as the truth lane"). The operator's identity, @tobiu, has none (`who_is_online`: wake `unsubscribed`).
- **How it reads.** The operator takes `wake off` to mean wakes don't work. Yet the fleet's open-work wakes are on: the gate was activated 2026-11-11 17:38Z, and the 2026-11-12 FM carries `neomjs/neo-agent-brain#1610`.
- **No other surface can show wake success either.** Every card's wake axis is `unknown`: "terminal delivery receipts live with the containerized delivery authority; not exposed yet".

## The Architectural Reality

Design authority: `ViewerWakeTelltale`'s JSDoc, "The chip is deliberately ALWAYS rendered … an operator glancing at chrome must be able to tell 'live' from 'nobody wired it' without a drill-in." For a viewer whose lane is never wired, that rule renders a permanent negative that answers no question the viewer has. The operator's report is the counter-evidence.

- `apps/agentos/util/ViewerWakeTelltale.mjs` (the derivation)
- `apps/agentos/view/fleet/cockpit/ViewerWakeTelltaleComponent.mjs`
- `apps/agentos/view/fleet/cockpit/StateProvider.mjs` (default `wake off` · `wake stream not started`)
- `apps/agentos/view/fleet/cockpit/ControlToolbar.mjs` (reference `viewer-wake-telltale`)

## The Fix

1. The chip renders only for a viewer whose push lane is armed, or has been alive this session. For a viewer the fleet server reports as not armed, the chip is absent, and the reason stays reachable in the diagnostics detail.
2. The header IA's wake-pill row records the change.

## Acceptance Criteria

- [ ] A viewer the fleet server reports as not armed sees no wake chip. An armed viewer still sees `wake live` / `wake off` with the title.
- [ ] Derivation specs cover armed and live, armed and down, and not armed.
- [ ] On the installed FM, the operator's cockpit shows no `wake off`.

## Out of Scope

- A fleet-level wake verdict in the chip (seats' delivery confirmed). It needs the plane to expose terminal delivery receipts first (Brain) and gets its own ticket when that lands.
- The cards' wake axis (`unknown`), which has the same dependency.

## Avoided Traps

- **A new state word for "not applicable".** Chrome labels are one status word pair (the header IA's label-language law); a third state word for an unarmed lane is noise in chrome.
- **Arming a push lane for the operator just to turn the chip green.** The operator is not a wake target.

## Related

`neomjs/neo-agent-brain#759` (open-work wakes), `neomjs/neo-agent-brain#1609` / PR #1610, the header IA (#533 / #542), #919.

Live latest-open + closed sweep: latest 20 open (58 open, red band) plus an all-state search (`wake telltale wake off viewer push`) at 2026-11-12T02:12Z. The only hit was the closed cockpit epic #10; no ticket owns this chip.
A2A in-flight claim sweep: latest messages, all states, at 02:10Z; no claim.
MC sweep: `cockpit activity feed empty after FM restart; banner text too long; wake shows off and never success`, 6 results; no prior decision.
Own-assignment sweep: 3 open (#917, #516, #424), none on this surface.

unowned-rationale: Ready for a maintainer. The not-armed signal's wire shape needs reading in the fleet server first; maintainers provide review.

Origin Session ID: c56bdafb-bd1d-4a67-b91d-3c37931e0abb
Retrieval Hint: "wake chip shows wake off forever: viewer push lane never armed for an unsubscribed viewer, ViewerWakeTelltale always rendered"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T02:15:57Z @neo-opus-ada added the `bug` label
- 2026-11-12T02:15:57Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T02:15:57Z @neo-opus-ada added the `ai` label
- 2026-11-12T02:15:58Z @neo-opus-ada added the `design` label

