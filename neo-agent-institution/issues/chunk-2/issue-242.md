---
id: 242
title: 'The perspective bar only moves a splitter: retire it for the drawer'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T09:34:24Z'
updatedAt: '2026-09-26T10:26:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/242'
author: neo-opus-ada
commentsCount: 0
parentIssue: 10
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
# The perspective bar only moves a splitter: retire it for the drawer

## Context

The operator, 2026-09-26, on the cockpit's top-left button group: *"overview and focus => just moving the splitter. review opening the side view, which you also get when selecting an agent card. the controls are utterly pointless this way => a waste of space."*

A headless walk of dev (1280×800, dark) confirms each part: Focus moves the fleet/strip splitter to 85/15, Overview moves it back, and Review opens the Agent Detail column, which a click on a card also opens. The same three layouts are listed a second time in the right rail's Perspectives drawer, with Apply and "Capture current layout".

## The Problem

The cockpit's most prominent spot holds three presets for a splitter the operator can drag directly, and a Review button that duplicates card selection. The bar came from `neomjs/neo#14616` (a preset library with "NL-verifiable switching"), and the drawer was later added for saved layouts, so the same choice now has two controls and the prominent one does the least.

## The Architectural Reality

- `apps/agentos/util/CockpitPerspectives.mjs`: `duties` declares Overview (`zones: {}`, the default split), Focus (`sizes: [0.85, 0.15]`) and Review (`sizes: [0.45, 0.55]`, `detailColumn: true`); `group()` builds the segmented bar.
- `apps/agentos/view/fleet/cockpit/Container.mjs`: the `fleet-control-bar` toolbar starts with `CockpitPerspectives.group()` and the `fleet-preset-error` line, then the state block (spine pill, wake telltale) and the actions (Reconnect, Start fleet).
- The Perspectives drawer (`view/fleet/perspectives/Container.mjs`) binds `dock.perspective.active` and `perspectives`; it applies and captures layouts through the same engine path as the bar.
- Card selection opens Agent Detail through the cockpit's selection flow, independent of any perspective.

## The Fix

1. Remove the bar and its refusal line from `fleet-control-bar`. The Perspectives drawer stays the one place to apply or capture a layout, and it shows a refused restore itself.
2. The control bar keeps its state block and actions. If Grace's design review agrees, the bar folds into the Fleet tab's header row, and the roster gets the row's height back.

## Acceptance Criteria

- [ ] AC-1 The cockpit renders no perspective button group; the drawer lists the layouts and applies them (the unit and NL arms that drove the bar move to the drawer).
- [ ] AC-2 The drawer keeps Overview, Focus and Review as saved layouts; a click on a card still opens Agent Detail.
- [ ] AC-3 A refused restore is reported in the drawer (unit arm).
- [ ] AC-4 The affected visual goldens are re-captured with the before/after in the PR, reviewed by the design owner.

## Out of Scope

- Retiring the Review duty (narrowed 2026-09-26 at intake). Review is the only declared perspective that docks the inspector, so five arms prove the cold-inspector seating through it. In the drawer it takes no space; whether it earns its place there is a design call.
- The state block's words (#239 for "connected, registry empty").
- The engine's perspective library (`src/dashboard/dock/persistence/PerspectiveLibrary.mjs`) — unchanged.

## Related

#10 (parent) · `neomjs/neo#14616` / `neomjs/neo#15004` (the bar's origin) · #13 (design conformance)

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): no claim on the cockpit bar. Memory Core sweep ("cockpit perspective bar Overview Focus Review"): the bar's origin (Grace's `neomjs/neo#15004` session, 2026-07-10) — no later decision to keep it. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("cockpit perspective bar retired Overview Focus Review drawer")`


## Timeline

- 2026-09-26T09:34:25Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T09:34:26Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T09:34:26Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:26Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:26Z @neo-opus-ada added the `design` label
- 2026-09-26T09:34:29Z @neo-opus-ada cross-referenced by #244
- 2026-09-26T09:34:30Z @neo-opus-ada cross-referenced by #245
- 2026-09-26T09:34:32Z @neo-opus-ada cross-referenced by #246
- 2026-09-26T09:34:33Z @neo-opus-ada cross-referenced by #247
- 2026-09-26T09:35:00Z @neo-opus-ada added parent issue #10
- 2026-09-26T10:38:05Z @neo-opus-ada cross-referenced by PR #251

