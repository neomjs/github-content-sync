---
id: 76
title: 'The first agent graduating into an empty roster renders an empty row: pooled cards after a rendered-empty list'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T00:48:43Z'
updatedAt: '2026-09-02T14:15:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/76'
author: neo-fable-clio
commentsCount: 1
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
closedAt: '2026-09-02T14:15:03Z'
---
# The first agent graduating into an empty roster renders an empty row: pooled cards after a rendered-empty list

Sub-issue of #10. Successor of #74 (the define-agent zone), found by `AddAgentJourneyNL` once the zone rendered again.

## Context

After the roster has rendered EMPTY (an authoritative zero-row fleet — the first-run state), the first agent that graduates into it shows as a title count with no card: `Fleet · 1 agents`, one `li` in `.fm-fleet-cards`, and the `li` is empty (live read through the witness, 2026-09-02 00:3xZ: `<li id="…__0" class="neo-list-item" style="height: 126px; width: 416px; position: absolute; …"></li>`).

## The Problem

The roster list pools one `AgentCard` instance per row index and re-seats it by reference (`createVdomReference` → `{componentId}`; `roster/List.mjs#createItemContent`). The engine's tree builder inlines a referenced child in full only while the child has no vnode; a child holding a vnode is sent as a placeholder ("move my existing node here" — `mixin/VdomLifecycle.mjs`: "Skip unmounted components. They will be expanded by the Parent's TreeBuilder"). When the roster drops to zero rows, the list items leave the DOM and the cards' DOM leaves with them — but nothing unmounts a child that leaves inside its parent's delta: the pooled cards (and every component inside them: the avatar `Image`, the verbs' Buttons) keep `mounted: true` and their vnodes. The next row re-seats a card that believes it is on screen, its reference becomes a placeholder, and the placeholder points at a node that no longer exists: an empty `li`.

## What was tried in the #74 lane, and why each shape failed (so nobody re-derives it)

1. **Destroy the pool on the render after an empty pass, recreate on first use** — the replacement card takes the retired card's index id; with the retired rows still fading out, the diff paints every row twice (6 cards for 3 rows in `FleetGridKeyboardA11y`'s roster reload, which is a clear-then-add inside one tick).
2. **Reset `vnode` / `mounted` on the pooled cards at the next `createItems` after an empty pass** (with and without a transition guard) — the same duplication: a card whose state is forgotten while its old node still stands is inlined beside that node.
3. **Reset on the list's `createItems` settle event when the store is still empty** — clean for the add-agent flow (the journey went green through Start), clean for the reload (no duplicates), but the AgentCard witness's `clear` → (a Neural Link round trip) → `add(4)` lost child components: the second card rendered without its avatar `Image` even though the reset walks `items` — the re-inline after a settled empty pass did not bring every child back. The truth is one level below the app: **a component removed inside its parent's delta must be told**, so that its next reference renders — an engine concern (`list.Component` pooled children, or the vdom lifecycle marking descendants unmounted when a subtree is removed), not a roster override.

## The Fix

- Engine-side first read: when a vdom update removes a subtree that contains referenced components, mark those components unmounted and drop their vnodes (the symmetric counterpart of the mount pass) so a later reference is inlined; or give `list.Component` an explicit "row removed" hook that does the same for its pooled instance. File the Engine leaf with the reproducer below and consume it here.
- If the engine change is not available in time, the roster may stop pooling across an empty state by construction (no pooled instance survives a zero-row render: destroy them on the settled empty pass, never on a refill) — only with the duplication trap above understood: the destroy must happen when the rows are really gone, never while they fade.

## Reproducer

`AddAgentJourneyNL` (Brain root bound): authoritative-empty roster → bootstrap CTA → S5 form → readback → `Fleet · 1 agents` → `.fm-fleet-cards .fm-agent-card` count 1 expected, 0 received. Minimal Neural Link shape: clear the FleetRoster store, let the empty render settle, add one row, read the `li`.

## Acceptance Criteria

- [ ] After a settled empty roster, the first added row renders its card with avatar, name and verbs (DOM read).
- [ ] The roster reload (clear-then-add inside one tick — `FleetGridKeyboardA11y`) still renders exactly one card per row, and the AgentCard witness's clear → add renders every card with its avatar.
- [ ] `AddAgentJourneyNL` green end to end; the cause named in the PR at the engine seam or the roster seam, with the receipt.

## Out of Scope

- The define-agent zone's materialization — #74.
- Any change to the animate plugin's fade timing.

## Related

Parent: #10. Predecessor: #74. The engine seams: `src/mixin/VdomLifecycle.mjs` (tree builder / `createVdomReference`), `src/list/Component.mjs` (pooled items), `src/list/plugin/Animate.mjs` (row removal with fade).

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-02T00:55Z — #74 (the zone) is the predecessor, nothing else names the roster pool or the empty-to-one transition; A2A lane-claims over the last hours cover Engine dock leaves, skills and Brain — none on this seam; Memory Core raw query on the pooled-card symptom returned nothing.

Origin Session ID: f353cda0-1b36-49e4-9b73-3f7aec1896e0

Retrieval Hint: `roster pooled AgentCard empty list vnode placeholder createVdomReference empty li after graduation`

📜 Clio

## Timeline

- 2026-09-02T00:48:45Z @neo-fable-clio added the `bug` label
- 2026-09-02T00:48:45Z @neo-fable-clio added the `agent-os` label
- 2026-09-02T00:48:45Z @neo-fable-clio added the `ai` label
- 2026-09-02T00:48:45Z @neo-fable-clio added the `regression` label
- 2026-09-02T00:49:36Z @neo-fable-clio cross-referenced by PR #77
- 2026-09-02T00:49:37Z @neo-fable-clio cross-referenced by #74
- 2026-09-02T01:18:39Z @neo-fable-clio cross-referenced by #18060
- 2026-09-02T01:23:48Z @neo-fable-clio cross-referenced by PR #18061
### @neo-fable-clio - 2026-09-02T01:24:22Z

Engine leaf filed and in review: neomjs/neo#18060 → PR neomjs/neo#18061 (`36bf90cb93`). The cause is one level below the roster, as suspected: `syncVnodeTree`'s unmount pass ran only for the component that received a vnode, one level deep — the list's empty render merged into a covering ancestor flight, the list never received its own vnode, and its pooled cards kept `mounted: true` with stale vnodes; every later re-seat by reference became a placeholder for a node the DOM no longer had. Measured live on the cockpit through the Neural Link (2026-09-02): `fleetRoster.clear()` → 0 `li`, 11 cards still mounted → `add(2 rows)` → two EMPTY `li`. So this is not first-run-only: any settled-empty roster followed by a refill blanks the grid.

The Engine fix runs the unmount pass for every synced component and recurses through removed subtrees; four-arm reproducer red at `46c441eaba`, green at the head. This ticket then closes by bumping the pinned `neo.mjs` SHA once the PR lands and re-running `AddAgentJourneyNL` — no roster-side workaround.

📜 Clio


- 2026-09-02T01:51:22Z @neo-opus-grace cross-referenced by PR #75
- 2026-09-02T09:29:15Z @neo-fable-clio cross-referenced by #78
- 2026-09-02T11:22:24Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-02T11:40:26Z @neo-fable-clio cross-referenced by PR #79
- 2026-09-02T12:24:10Z @neo-fable-clio cross-referenced by #80
- 2026-09-02T13:14:36Z @neo-fable-clio cross-referenced by PR #18086
- 2026-09-02T14:15:03Z @tobiu referenced in commit `2cdf5c7` - "Merge pull request #79 from neomjs/agent/76-engine-pin-bump

chore(deps): bump the engine pin to dev@5ae9358063 — pooled children unmount, frame-origin window geometry (#76)"
- 2026-09-02T14:15:03Z @tobiu closed this issue
- 2026-09-02T14:46:01Z @neo-fable-clio cross-referenced by #81
- 2026-09-02T14:52:33Z @neo-fable-clio cross-referenced by PR #82
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90

