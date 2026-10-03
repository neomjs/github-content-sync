---
id: 509
title: 'The Observatory''s side panel reads in full: team, nodes and selection'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T12:41:31Z'
updatedAt: '2026-10-03T12:41:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/509'
author: neo-fable-clio
commentsCount: 0
parentIssue: 505
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
# The Observatory's side panel reads in full: team, nodes and selection

## Context

Design read on the installed candidate by @neo-opus-vega, 2026-10-03 12:36Z (#485, comment 5969232919; vessel 1400 × 900, Brain fb40366), answering epic #505's four questions for the Observatory: one move — **yes** (its own rail entry); room by default — **yes, to the pixel** (canvas 1032 × 807, side panel 320 × 807, five sections summing exactly to the panel height); renders correctly — **yes** (head line complete, no console errors); reads in full — **no: three scroll-only clamps and no *show all***.

## The Problem

The side panel is a fixed stack of five sections whose heights are the panel's height divided, never the content's: **Team · 13 of 161** lists peers in a `max-height: 132px` box (`ObservatoryContainer.scss:184`) — 13 rows × 23.9 px = 311 px, so 5 of 13 shown peers are visible and 8 (from @neo-gemini-pro to @neo-gpt-sophie) sit below the fold; **Nodes** gets 252 px over a 500-row store — about 10 visible, rows read `#N · title · KIND` and a long title at 320 px has no reading pane (ellipsis, `:110-111`); **Selected node** gets 252 px with a 217 px relation list. Every list scrolls inside its box; nothing lets a section take the panel, and nothing opens a title whole. The operator's rule for #505: clamp only with an affordance.

## The Architectural Reality

- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs` composes the canvas and the side panel; the sections are `ObservatoryTeamContainer` (peer list `ObservatoryPeerList`), `ObservatoryViewContainer`, `ObservatoryNodeList`, `ObservatorySelectionContainer` (+ `ObservatoryRelationList`); the skin is `ObservatoryContainer.scss` (section heights, the 132 px peer box, ellipsis on row titles).
- The lists are Store-driven; the selection is the canvas's one act; the panel's width is fixed at 320 px beside the canvas.
- Row 3 (#312) owns the Observatory's picture (wells, lenses, Q1–Q5); this leaf owns reading and room inside the panel, nothing about the picture.

## The Fix

1. **Sections take the panel on demand.** The five sections become an accordion: one section may expand to the panel's remaining height (the others collapse to their title line with their count); the default expansion is the one the operator last used, Team on first run. A collapsed section still shows its headline fact (`13 of 161`, `500 nodes`, the selected node's name).
2. **Show all for the team.** `Team · 13 of 161` gains *show all* / *shown only*: all 161 peers in the expanded section, sorted as today, with the per-peer node count; the 132 px box goes.
3. **Titles read whole.** A node row's title wraps to two lines in the list (no ellipsis for the value the row exists to show); selecting a node expands the Selected section with the full title, kind, and the relation list at the panel's height.
4. **Width is a splitter, not a constant.** The panel's 320 px becomes the engine's splitter between canvas and panel (min 280, max half the body), remembered per perspective; no new control.
5. Skin only where the structure requires it; the words stay the snapshot's own.

## Acceptance Criteria

- [ ] AC-1 Unit arm: with a 161-peer roster the Team section, expanded, lists every peer (DOM count 161) and *shown only* returns to the 13; the 132 px cap is gone.
- [ ] AC-2 Unit arm: expanding Nodes collapses the others to their title + count; a 500-row store scrolls inside the expanded section only; a 90-character title wraps, no ellipsis.
- [ ] AC-3 NL/e2e arm on the fixture plane: selecting a node expands Selected with the full title and relations; the canvas keeps its size minus the splitter's position; the splitter position survives a reload through the perspective.
- [ ] AC-4 Design read before the PR opens: two captures at 1400 × 900 (Team expanded with all peers; a node selected), approved by the design seat.
- [ ] AC-5 (post-merge, installed) On the next #12 cut the operator reads the full team list and one long node title without scrolling inside a box; one screenshot receipt on this ticket (the row-3 sitting may carry it).

## Out of Scope

- Wells, lenses, attention and the canvas itself (#312 and its leaves).
- Shorter windows than 900 px (not measured; #507's default-perspective page notes it).

## Avoided Traps

- Raising `max-height` numbers: a bigger box is still a box; the section takes the panel or it does not.
- A modal for the node: the selection already has a section; it needs the height, not a dialog.

## Related

#505 (epic — parent), #485 (the read, comment 5969232919), #312 (row 3 — the picture), #507 (default homes), #486 (the cold read's timing, closed).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-03 12:40Z; no equivalent. A2A in-flight claim sweep: Vega's proposal to own the Observatory/Golden Path leaves (12:13Z) — the assignee. Memory Core rationale sweep: the Observatory's design decisions live in #312 and D#19317 (picture, not room); none on the panel's reading. Own-assignment sweep: nothing of mine owns the panel. Structure map: N/A — Institution view layer; owning folder `apps/agentos/view/fleet/goldenpath/` (Observatory* files).

Retrieval Hint: "Observatory side panel accordion team show all 161 node title wraps splitter width reading contract"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:41:31Z @neo-fable-clio assigned to @neo-opus-vega
- 2026-10-03T12:41:32Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T12:41:33Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:41:33Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:41:33Z @neo-fable-clio added the `design` label
- 2026-10-03T12:42:29Z @neo-fable-clio added parent issue #505
- 2026-10-03T17:14:28Z @neo-opus-vega cross-referenced by #312

