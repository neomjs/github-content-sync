---
id: 508
title: 'System service cards read in full: no clipped status or diagnosis'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-03T12:32:25Z'
updatedAt: '2026-10-03T12:33:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/508'
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
# System service cards read in full: no clipped status or diagnosis

## Context

Operator screenshot of the installed Fleet Manager's System view, 2026-10-03 14:19 CEST (relayed by @neo-gpt-sophie, who holds the file and the native accessibility read): the service cards' observed-age/header text, classification/sample text, heap-observation reason and diagnosis run past the card edges, while the screen has substantial unused space. The accessibility tree of `app://neo/apps/agentos/index.html#/system` contains the complete strings — the payload is present, the rendering loses it. The operator's words: one example of MANY cut-off views. Leaf of epic #505 (every important view reachable, roomy, correct, readable); System is the inventory's last entry by daily weight and the first with a screenshot in hand.

## The Problem

The System list (`apps/agentos/view/system/List.mjs`) renders one card grammar per service — head line `[name] [compose id] [state word] [observed age]`, three facts (memory · class · restart churn) as a definition list, and the orchestrator's diagnosis line — over a responsive grid (`resources/scss/src/apps/agentos/system/List.scss`: `repeat(auto-fill, minmax(280px, 1fr))`, `overflow-wrap: anywhere` on the fact values, a container query under 300 px). On the installed build the text still escapes the cards: the head line's spans and the diagnosis line are not the facts' `dd`s, so whatever wrap rule the facts have does not reach them, and a 280 px minimum column packs cards where the screen could give each one three times the width. The view answers the operator's four questions badly on two: room by default (narrow columns on a wide screen) and correctness (clipped text).

## The Architectural Reality

- Card grammar and words: `List.mjs` `createItemContent` (head spans `fm-plane-key` / `fm-plane-id` / `fm-plane-word` / `fm-plane-seen`, the `dl` facts, the `fm-plane-diag-*` spans); every word is the snapshot's own (`observedLine`, `memoryLine`, `classLine`, `churnLine`, `diagnosisLine`).
- Skin: `List.scss` grid + container query; `Container.scss` for the view frame. The installed build is e1a9dbe (engine 82bc615); the capture names the exact CSS that clips once Sophie's trace lands on this ticket.
- Contract: no CARD-CONTRACT row owns the service card (that file is the roster card's); this leaf's §The Fix is the service card's reading contract until a design page exists.

## The Fix

1. **Nothing on a service card clips.** The head line wraps as a line of words (state word and observed age may drop to a second line); the diagnosis is a paragraph (`p`-class block, `overflow-wrap: anywhere`, no `nowrap`); fact values keep their wrap. A card's height follows its content; no fixed heights.
2. **Width follows the screen.** The grid's minimum column grows with the container (≥ 360 px at the operator's window, 1 column under 300 px as today), so on a wide screen a few readable cards replace many clipped ones; the unused space is spent on the text.
3. **Order of words kept**: the state word first, the age after it, the diagnosis last — the grammar stays, only its room changes.
4. The operator's screenshot and Sophie's accessibility read are the before; one capture at the same window size is the after; a visual golden of the System view is added if the suite lacks one.

## Acceptance Criteria

- [ ] AC-1 Unit arm: a service row with a 240-character diagnosis, a long compose id and a long classification renders every string whole (no element wider than its card; `scrollWidth <= clientWidth` on card, head line and diagnosis line).
- [ ] AC-2 Visual arm: the System view at 1600 px wide shows cards whose minimum column is ≥ 360 px; at 290 px one column; no text past a card edge in either.
- [ ] AC-3 The cause as named by the installed trace is on this ticket before the PR opens (which rule clipped: a `nowrap`, a fixed height, a flex item without `min-width: 0`, or the grid's minimum), and the fix removes it rather than hiding it.
- [ ] AC-4 (post-merge, installed) On the next #12 cut the operator's System view shows every service card in full at his window size; one screenshot receipt on this ticket.

## Out of Scope

- New facts on the card or a redesign of the grammar.
- The System view's placement in the rail (#507 decides homes).

## Avoided Traps

- Truncating with an ellipsis and a tooltip: the words are the content; a `title` attribute is not an affordance.
- Fixing the symptom with a larger fixed card height: the card's height follows its content.

## Related

#505 (epic — parent), #507 (default homes), #312 (the Observatory, the other operating-picture view), #12 (the installed cut that carries AC-4).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-03 12:31Z and a keyword search ("System view service card", open) — no equivalent. A2A in-flight claim sweep at 12:31Z: Sophie's QC offer only, no competing claim. Memory Core rationale sweep: no prior decision on the service card's room. Own-assignment sweep: none of my open tickets owns the System view. Structure map: N/A — Institution view layer; owning folder `apps/agentos/view/system/`.

handoff: @neo-gpt-sophie holds the evidence (screenshot + accessibility read) and the QC; she builds it if she claims it, otherwise the first free peer through a planner.

Retrieval Hint: "System view service cards clipped diagnosis head line wrap grid minimum column reading contract"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:32:26Z @neo-fable-clio added the `bug` label
- 2026-10-03T12:32:26Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:32:26Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:32:27Z @neo-fable-clio added the `design` label
- 2026-10-03T12:33:06Z @neo-fable-clio added parent issue #505
- 2026-10-03T12:33:08Z @neo-fable-clio assigned to @neo-gpt-sophie

