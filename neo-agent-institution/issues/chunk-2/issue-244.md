---
id: 244
title: 'Home gets a live canvas, at least at the portal hero''s bar'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-09-26T09:34:28Z'
updatedAt: '2026-09-26T09:34:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/244'
author: neo-opus-ada
commentsCount: 0
parentIssue: 9
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
# Home gets a live canvas, at least at the portal hero's bar

## Context

The operator, 2026-09-26: *"the home view looks boring. e.g. the portal app uses the home canvas. this is the LOWEST bar i can think of."*

The Home view today is one HTML string: an eyebrow, a two-line headline ("Mission control for a cross-model AI engineering team.") and a lede that tells the operator to click Fleet — on a flat background, with nothing live and nothing to act on.

## The Problem

Home is the rail's first destination (Fleet is the default view) and the product's welcome. The engine already ships the pattern the operator named: the portal's hero draws the "Neural Swarm" on the canvas worker (`Portal.view.home.parts.hero.Canvas` → `Portal.canvas.HomeCanvas`), zero-allocation, theme-aware, reacting to the pointer. The cockpit already runs a canvas-worker renderer too (#230's Observatory). Home uses neither.

## The Architectural Reality

- `apps/agentos/view/Viewport.mjs`: the Home tab item is `{header: {iconCls: 'fa-solid fa-house', route: '/home', text: 'Home'}, ...}` with the hero as an `html` string (`agent-welcome-h1`, `agent-welcome-lede`).
- The engine pattern: `src/app/SharedCanvas.mjs` (App-worker controller: offscreen transfer, resize observation, pointer bridging) + a `Neo.canvas.Base` renderer imported into the canvas worker by `rendererImportPath`. `apps/portal/view/home/parts/hero/Canvas.mjs` (44 lines) + `apps/portal/canvas/HomeCanvas.mjs` is the reference; `apps/agentos/canvas/` holds the cockpit's renderers and `fmPalette.mjs`.
- #237 governs data: nothing in the cockpit seeds invented data. A decorative field claims nothing; anything that reads as fleet state must come from a real read.

## The Fix

1. `apps/agentos/view/home/` gets a Home view class: the hero copy over a `SharedCanvas` component.
2. `apps/agentos/canvas/HomeCanvas.mjs` renders an ambient field in the FM palette, at least at the portal hero's bar (motion, depth, pointer response, light and dark, `prefers-reduced-motion` honoured).
3. When the cockpit has real reads, the field may reflect them (one mote per rostered agent, a pulse per real activity event); with no reads it stays ambient and claims nothing.
4. The lede ends in actions, not directions: "Open the fleet" and, in a shell without a plane, "Connect a plane".

## Acceptance Criteria

- [ ] AC-1 Home renders the canvas behind the hero in both themes; the renderer draws on the canvas worker (NL or e2e arm reads its stats), and idles under `prefers-reduced-motion`.
- [ ] AC-2 With no reads, the field shows no counts or agent identities; with a live roster, any per-agent mark matches the roster's length (unit arm on the scene input).
- [ ] AC-3 The hero's actions route to Fleet and, in a shell without a plane, mount the plane-setup card.
- [ ] AC-4 Visual goldens for Home in light and dark, reviewed by the design owner.

## Out of Scope

- The Chat view's placeholder (its own destination work).
- An onboarding flow beyond the two actions.

## Related

#9 (parent: keeper views · nav) · #10 · #237 (no seeded data) · #230 (the cockpit's canvas-worker precedent)

unowned-rationale: design-led and claimable — it needs a design pass (Grace holds design review on #10) and a canvas-worker renderer; not started by its author, who holds #241 and #242.

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): no claim on Home. Memory Core sweep ("Agent OS home view hero canvas"): no prior decision. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("Agent OS Home view canvas hero portal Neural Swarm")`

## Timeline

- 2026-09-26T09:34:29Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T09:34:29Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:29Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:29Z @neo-opus-ada added the `design` label
- 2026-09-26T09:35:03Z @neo-opus-ada added parent issue #9

