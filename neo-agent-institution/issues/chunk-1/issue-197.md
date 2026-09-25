---
id: 197
title: The cockpit bar's Start fleet ends 10 px from the edge; the header's theme switch 20
state: CLOSED
labels:
  - bug
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-25T11:44:29Z'
updatedAt: '2026-09-25T12:05:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/197'
author: neo-fable-clio
commentsCount: 0
parentIssue: 13
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T12:05:03Z'
---
# The cockpit bar's Start fleet ends 10 px from the edge; the header's theme switch 20

## Context

Operator, 2026-09-25, looking at the installed shell: *"right-side alignment of items: the switch theme button has a bigger gap to the right than start fleet."* Measured on `dev@d2d9f86` in the browser (Playwright, CSS px, identical at 1000 / 1400 / 2000 wide):

| control | right edge | gap to the viewport edge | host | host inset |
|---|---|---|---|---|
| theme switch (`.agent-theme-button`) | 1380 | **20** | `.agent-top-toolbar` (full width) | `padding: 0 20px` |
| Start fleet (`.fm-fleet-start`) | 1390 | **10** | `.fm-cockpit-bar` (full width, y 50–110) | `padding: 6px 10px` |
| pane header × (`.fa-times`) | 1386 (box), ink ≈ 1380 | 14 | `.neo-tab-header-toolbar`, ends at 1380 | `margin-right: -6px`, deliberate (ink on the flush line, Viewport.scss:91) |
| Hide benched (`.fm-roster-filter`) | 1360 | 40 | `.fm-fleet-controls`, ends at 1364 | pane content, 20 inside the pane edge |
| right rail (`.neo-dashboard-dock-edge-rail-right`) | x 1380–1400 | — | from y 110 | 20 wide |

## The Problem

The frame has one right gutter: 20 px — the rail's width, the header's inset, the line the pane headers' ink ends on. The cockpit bar is the one full-width chrome row that ends 10 px from the edge, so its last control (Start fleet) sits 10 px right of the theme switch above it and of the × below it. The eye reads a ragged right edge on the three rows that should share a line.

## The Architectural Reality

- `resources/scss/src/apps/agentos/Viewport.scss:193-204` — `.agent-top-toolbar { padding: 0 20px }` (a literal; the FM spacing scale `--fm-space-1..4` = 4/8/12/16 px has no 20).
- `resources/scss/src/apps/agentos/fleet/cockpit/Container.scss:93-96` — `.fm-cockpit-bar { padding: 6px 10px; gap: var(--fm-space-2) }`; its container queries (`:281`, `:308`) touch flex-wrap and label collapse, not the padding.
- The pane-header × overhang is designed (ink-on-line, Viewport.scss:91-94) and already lands on the 20 px line; the roster's controls are pane content with their own 20 px inner gutter — both stay.

## The Fix

One token for the frame gutter, `--fm-frame-gutter: 20px`, beside the spacing scale in `Viewport.scss`; `.agent-top-toolbar` reads it instead of its literal; `.fm-cockpit-bar` becomes `padding: 6px var(--fm-frame-gutter)` (its 560 px media rule keeps `--fm-space-3`). Start fleet then ends at 1380 with the theme switch and the pane headers. The visual goldens that show the bar move by 10 px and are re-rendered in the same leaf with a by-eye read (content vs chrome); the baseline stamp follows.

## Acceptance Criteria

- [ ] At 1000, 1400 and 2000 px wide, `.agent-theme-button` and `.fm-fleet-start` end at the same x (viewport width − 20), measured by the probe above.
- [ ] `.agent-top-toolbar` and `.fm-cockpit-bar` read one gutter token; no 20 px literal remains for the frame gutter in `src/apps/agentos`.
- [ ] `npm run test-visual` alone: every golden that moved is updated in this leaf and named (which, why); nothing else drifts.
- [ ] The baseline stamp is re-issued after staging; `check-visual-baselines` exits 0 on the pushed head.

## Out of Scope

- The perspective switch's noun (tab strip vs segmented control), the wake telltale's two-row stack, the roster lane-line ellipsis, the ◇ badge — the other rows of the design pass on #13, each its own leaf.
- The engine's tab-header button geometry.

## Related

#13 (parent) · #10 · #194 (the theme map, without which none of this reached the shell) · the design pass read on #13 (issuecomment-5831663831)

Live latest-open sweep: all 15 open issues checked at 2026-09-25 11:43Z; no equivalent. A2A claim sweep: no claim on the cockpit bar. Memory Core: no prior decision on the frame gutter.

Origin Session ID: 0fbfde3a-e817-4859-9351-2269eabdda9a
Retrieval Hint: "fm frame gutter 20px cockpit bar Start fleet theme switch right alignment"

## Timeline

- 2026-09-25T11:44:29Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-25T11:44:30Z @neo-fable-clio added the `bug` label
- 2026-09-25T11:44:31Z @neo-fable-clio added the `ai` label
- 2026-09-25T11:44:32Z @neo-fable-clio added the `design` label
- 2026-09-25T11:44:43Z @neo-fable-clio added parent issue #13
- 2026-09-25T11:52:02Z @neo-fable-clio cross-referenced by PR #198
- 2026-09-25T11:59:41Z @neo-fable-clio cross-referenced by #199
- 2026-09-25T12:05:03Z @tobiu referenced in commit `2291731` - "Merge pull request #198 from neomjs/fix/197-frame-gutter

fix(cockpit): the cockpit bar ends its last control on the frame gutter (#197)"
- 2026-09-25T12:05:03Z @tobiu closed this issue
- 2026-09-25T12:20:01Z @neo-fable-clio cross-referenced by #201
- 2026-09-25T13:56:26Z @neo-fable-clio cross-referenced by #208

