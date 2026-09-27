---
id: 294
title: 'The keeper rail''s active tab is a pressed icon tile, not a strip on the rail''s right edge'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-27T13:52:45Z'
updatedAt: '2026-09-27T13:55:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/294'
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
closedAt: '2026-09-27T13:55:05Z'
---
# The keeper rail's active tab is a pressed icon tile, not a strip on the rail's right edge

## Context

Operator, 2026-09-27 13:3xZ (release-goal review): "the left-side FM nav ⇒ team replaced items with icons (good), but kept the tab strip at the right side. this looks like SHIT. instead of giving the active tab header for left-side icons a nice pressed state for their tabs." Measured on dev `635afe7` (engine pin `942b43c8b8`), FM dev preview at 800×600: the keeper rail is 48 px wide, its six buttons 48×40 with `background: transparent` in every state; the active button (`pressed`) differs only by glyph ink, and a `neo-tab-strip` (2 px, the engine's `tab.Strip`) runs down the rail's right edge at x = 48 carrying the `neo-active-tab-indicator`.

## The Problem

`apps/agentos/view/Viewport.mjs` builds the keeper nav as a `tab.Container` with `tabBarPosition: 'left'`; the engine's default `useActiveTabIndicator: true` renders the strip, and the FM sheet (`resources/scss/src/apps/agentos/Viewport.scss`, the tab-family block) styles that indicator (`--tab-strip-height: 2px`, `--tab-indicator-background-color-active: var(--fm-signal)`, the clip that trims it). Since #267 made the rail an icon rail, the strip is the only mark of the active place besides glyph ink — a hairline beside a 16 px glyph, and it doubles the rail's own `border-right` hairline. An icon rail's active place is a pressed tile: the icon sits on a surface, the surface is the state.

## The Fix

1. `useActiveTabIndicator: false` on the keeper `tab.Container` — the strip is gone from the shell nav (the south content tabs and the detail's inner strip keep theirs).
2. The keeper rail's buttons become tiles: a fixed square inside the 48 px rail with the rail's rhythm as margin, a radius, transparent at rest; hover = the panel surface; `pressed` = the panel surface with a hairline ring (`--fm-line`) and the glyph in `--fm-signal` — declared through the engine's `--tab-button-*` tokens on the rail toolbar (the same seam the rail already uses for text/glyph colour), never by re-painting the button element.
3. The indicator-only declarations for the keeper rail (the `neo-dock-left` clip rule) go; the horizontal strips are untouched.

## Acceptance Criteria

- [ ] AC-1: the shell's keeper rail renders no `neo-tab-strip`; the south content tabs and the detail's inner strip still do (unit arm on the viewport definition + the existing tab-family arms green).
- [ ] AC-2: the active keeper tab's button carries a visible pressed surface (a computed background different from the rail's, a ring, signal glyph); the inactive ones stay transparent (component arm reading computed styles).
- [ ] AC-3: before/after captures on the PR at the FM preview; the visual baselines the stamp covers are re-captured where the rail is in frame.

## Out of Scope

- The cockpit's right-hand dock rail (Agent detail · Perspectives · Add agent · Wake routes) — rotated chips from #268; a separate call if the operator wants icons there too.

## Related

#267, #268 (the icon rail), #9 (the cockpit nav epic).

Live latest-open sweep at 2026-09-27T13:48Z: no open Institution issue on the keeper rail's active state (#247 is the reading strip's panes; #9 the epic). A2A: lane claimed 13:48Z to AGENT:* with the write-surface, no competing claim. Structure map: `apps/agentos/view/Viewport.mjs` + the `Viewport.scss` tab-family block; no new file.

Origin Session ID: 574ae0b8-b8d0-40d3-8cf6-1693ec48674a

## Timeline

- 2026-09-27T13:52:45Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-27T13:52:47Z @neo-opus-vega added the `enhancement` label
- 2026-09-27T13:52:47Z @neo-opus-vega added the `agent-os` label
- 2026-09-27T13:52:47Z @neo-opus-vega added the `design` label
### @neo-opus-vega - 2026-09-27T13:55:05Z

Duplicate of #267 (reopened by @neo-opus-grace 13:51Z) — the fix landed in @neo-opus-ada's PR #295 (Resolves #267): `useActiveTabIndicator: false` on the shell + a pressed tile, both skins, goldens re-captured. My 13:48Z claim was earlier; redoing finished work is the waste the operator named, so this closes and my local edits are reverted. Measurement ported to #267.

- 2026-09-27T13:55:05Z @neo-opus-vega closed this issue
- 2026-09-27T13:55:07Z @neo-opus-vega cross-referenced by #267

