---
id: 85
title: 'The roster''s health legend clips in a narrow fleet pane: the Review preset hides "benched / offline" behind overflow instead of wrapping or folding'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-02T15:49:47Z'
updatedAt: '2026-09-04T12:05:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/85'
author: neo-fable-clio
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
closedAt: '2026-09-04T12:05:54Z'
---
# The roster's health legend clips in a narrow fleet pane: the Review preset hides "benched / offline" behind overflow instead of wrapping or folding

Sub-issue of #10 (the cockpit UI/UX epic). Seen on 2026-09-02 during the pre-rehearsal design pass on engine pin `4e0e9dd8c3` (PR #82 + PR #83 merged), both themes.

## Context

The fleet pane's head row (`.fm-fleet-head`: the `Fleet · 11 agents · static roster` title, then the health legend `.fm-health-bar` with seven `.fm-health-swatch` items — working / idle / wedged / rate-limited / unobserved / external harness / benched · offline) is one `nowrap` flex row with `overflow-x: hidden`. In the **Review** preset the fleet pane is 896px wide (1280×720 viewport, the inspector docked right), and the row can no longer hold the legend.

**Measured** (Browser pane, Review preset, `1280×720`): `.fm-fleet-head` clientWidth 864 / scrollWidth 1017; the last swatch (`0 benched / offline`) ends at x = 1083 while the row's right edge is at x = 930 — the seventh state is fully hidden and the sixth (`3 external harness`) is cut at its trailing edge, in dark and in light. In the Overview and Focus presets the row is wide enough and all seven show.

## The Problem

A legend that silently drops its last states at a narrower pane width says "these states do not exist here" — the opposite of what a legend is for — and it happens in a shipped preset one click from boot. Nothing scrolls, nothing wraps, nothing folds; the clip is invisible unless the reader knows the seventh state.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/Container.mjs` — the head row (`fm-fleet-head`) hosts the title and the `fm-health-bar` (`fm-animate-counts`, the `fm-health-nominal` level class).
- `resources/scss/src/apps/agentos/fleet/roster/Container.scss` — the head row's `nowrap` + `overflow: hidden`; the card below already answers its OWN width through `@container` (ADR 0029: card-owned responsiveness), the head row does not yet.
- The fleet pane's width is dock-owned (preset, splitter, tear-out vessel), so a viewport media query is the wrong instrument; the pane or the head must be the query container.

## The Fix

Let the head answer its own width, in this order of preference:
1. **Wrap:** `.fm-health-bar` gets `flex-wrap: wrap` with a `row-gap` on the vocabulary's ladder, and the head row wraps the bar under the title once the pane's content box drops below the width that holds all seven (measure the breakpoint; ~1000px content at this type size). The legend keeps its seven-state vocabulary at every width and costs one extra line only when narrow.
2. If the extra line is unwanted at the narrowest band (a tear-out vessel), fold **zero-count** swatches into a single `· 4 more` chip at that band only — never drop a non-zero state.

Cards, the sort/filter bar and the presets stay untouched.

## Acceptance Criteria

- [ ] AC-1 In the Review preset at 1280×720, every one of the seven legend swatches is fully inside the head row's box (`scrollWidth === clientWidth` on `.fm-fleet-head`, the last swatch's right edge ≤ the row's), both themes, read by eye.
- [ ] AC-2 Overview and Focus render the legend on one line as today (no regression at the wide widths) — the synthesis / visual goldens that cover the head, if any, re-captured with a design read; otherwise a component arm that measures the head at 896px and 1200px pane widths.
- [ ] AC-3 `check-visual-baselines` stamped after staging; unit + NL battery green with only the standing reds.

## Out of Scope

- The legend's vocabulary and its count animation.
- The card grid's own width modes (#15536 settled them).

## Related

Parent: #10. #23 (cockpit header IA and responsiveness — the top bar, not this row). ADR 0029 (card-owned responsiveness via `@container`).

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-02T15:52Z (#78 … #9); a `legend roster header overflow` search returns nothing; #23 covers the top bar's IA, not the fleet head. A2A: no claim on the roster head in the window. Structure-map gate: N/A. Structural pre-flight: N/A — SCSS and possibly one container-query block, no new `.mjs`.

Origin Session ID: 91f83b9c-df95-4f72-a68f-d33f470792ac

Retrieval Hint: "fleet head health legend clips overflow hidden Review preset wrap container query"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 91f83b9c-df95-4f72-a68f-d33f470792ac

## Timeline

- 2026-09-02T15:49:49Z @neo-fable-clio added the `bug` label
- 2026-09-02T15:49:49Z @neo-fable-clio added the `agent-os` label
- 2026-09-02T15:49:49Z @neo-fable-clio added the `ai` label
- 2026-09-02T17:12:50Z @neo-fable-clio cross-referenced by PR #86
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T09:55:22Z @neo-fable-clio cross-referenced by PR #91
- 2026-09-04T09:58:01Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T10:08:19Z @neo-fable-clio referenced in commit `c62a0a0` - "test(agentos): the synthesis goldens follow the two-line fleet head at the harness width (#85)"
- 2026-09-04T10:08:33Z @neo-fable-clio cross-referenced by PR #93
- 2026-09-04T10:28:27Z @neo-fable-clio cross-referenced by PR #94
- 2026-09-04T11:46:02Z @neo-fable-clio referenced in commit `7f51a0e` - "fix(agentos): the fleet head wraps its health legend under the title instead of clipping its last states (#85)"
- 2026-09-04T11:46:02Z @neo-fable-clio referenced in commit `45211e8` - "test(agentos): the synthesis goldens follow the two-line fleet head at the harness width (#85)"
- 2026-09-04T12:05:54Z @tobiu referenced in commit `78c140f` - "Merge pull request #93 from neomjs/agent/85-fleet-head-legend-wrap

fix(agentos): the fleet head wraps its health legend under the title instead of clipping its last states (#85)"
- 2026-09-04T12:05:54Z @tobiu closed this issue

