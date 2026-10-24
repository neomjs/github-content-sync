---
id: 790
title: 'The Activity stream has no keyboard path: rows never take focus, arrows do nothing'
state: CLOSED
labels:
  - bug
  - accessibility
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-23T15:12:43Z'
updatedAt: '2026-10-24T12:41:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/790'
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
closedAt: '2026-10-24T12:41:02Z'
---
# The Activity stream has no keyboard path: rows never take focus, arrows do nothing

## Context

Operator report (2026-10-23, on his installed Fleet Manager): keyboard navigation inside the Activity stream does not work properly. Code read at `origin/dev` 024a7a5586: it was never wired — the list is declared without focusable rows and without selection, so there is nothing for the arrow keys to move.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-23 14:5xZ; no equivalent (#566, closed, skinned the recipient avatar and the new-events pill). A2A claim sweep (last 14 broadcasts): none. Memory Core rationale sweep: no decision that the stream is pointer-only recovered. Own-assignment sweep: #505/#507 are the parents.

## The Problem

`apps/agentos/view/fleet/activity/Container.mjs:345-357` builds the stream as a `Neo.list.Buffered` (`fm-activity-list`) with `disableSelection: true` and `itemsFocusable: false`, items being `RowContainer`s. With no focusable items the engine's Navigator (`src/list/Base.mjs`, built behind `hasNavigator`) never subscribes, so ArrowUp/ArrowDown, Home/End and Enter have no target; Tab lands on whatever chips inside a row are buttons and skips rows that have none. The observer mode already has a pointer-only verb — a message row opens beside the list (`activity/Container.scss:93-95` `.is-observing … .is-message {cursor: pointer}`, the detail and `stream-splitter` at `Container.mjs:526-534`) — with no keyboard equivalent, and no open-row paint exists in the stream's SCSS at all.

#505's question 3 ("no controls that drop focus") and the cockpit's keyboard contract on the roster (`roster/SelectionModel.mjs`: arrows move the Navigator's focus, Enter selects) make the stream the odd surface out.

## The Architectural Reality

- `Neo.list.Base`: `itemsFocusable` + `navigator` give arrow-key focus for free; `selectionModel` is separate — the stream can stay unselected while rows take focus.
- The open-record state already exists in the container (`splitter.hidden = !record`); Enter on the focused row is its keyboard activation, the same carve-out the roster uses (`SelectionModel.mjs:57-64`).
- The open row's paint should be the cockpit's selection language (the roster card's tint; the Memories leaf aligns its ring to it), not a new one.

## The Fix

In `activity/Container.mjs`: `itemsFocusable: true` with the list's Navigator (keep `disableSelection: true` unless the open-record state is better expressed as a single selection — the owner's call, stated in the PR); Enter on a focused message row opens it beside the list in observer mode exactly as the click does, Escape closes it; the open row carries `aria-current` (or the selection class) and the roster's selection paint in `activity/Container.scss`. No change to the stream's data path.

Design authority: #505 question 3 and the roster's keyboard contract; the operator's 2026-10-23 report.

## Acceptance Criteria

- [ ] AC-1 (red-first, NL/e2e on a source build): with the stream focused, ArrowDown moves row focus (the Navigator's focused item advances), Enter on a message row in observer mode opens it beside the list, Escape closes it — red on the current source (focus never enters a row).
- [ ] AC-2: the open row's paint equals a selected roster card's (computed background and border), light and dark.
- [ ] AC-3: Tab order: list → rows (one stop) → the row's chips only while a row is focused, or chips excluded from the sequential order (the owner states which, with the reason).
- [ ] AC-4 (post-merge): the operator walks the stream by keyboard on the installed app.

## Out of Scope

Virtual-scroll focus retention across buffer recycling beyond what `Neo.list.Buffered` already guarantees (a separate leaf if it shows); the mailbox detail's own keyboard handling.

## Avoided Traps

A custom keydown handler on the container (the engine's Navigator is the mechanism); making every chip a tab stop (a row of chips becomes a maze); inventing a stream-only open-row color.

## Related

#505 (parent) · #566 (the stream's avatar and pill skin) · the roster `SelectionModel.mjs` (the keyboard contract to mirror) · neomjs/neo `src/list/Base.mjs` (`itemsFocusable`, `navigator`).

handoff: unowned — a builder seat claims after the v13.3.1 cut; the cockpit steward routes.

Origin Session ID: d0801374-2558-4c56-97d0-ba4a719149a3

Handoff Retrieval Hints: `query_raw_memories("activity stream itemsFocusable disableSelection navigator keyboard observer message row")`; the operator's 2026-10-23 FM report.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session d0801374-2558-4c56-97d0-ba4a719149a3

## Timeline

- 2026-10-23T15:12:44Z @neo-fable-clio added the `bug` label
- 2026-10-23T15:12:45Z @neo-fable-clio added the `accessibility` label
- 2026-10-23T15:12:45Z @neo-fable-clio added the `agent-os` label
- 2026-10-23T15:12:45Z @neo-fable-clio added the `ai` label
- 2026-10-23T15:12:45Z @neo-fable-clio added the `design` label
- 2026-10-23T15:13:40Z @neo-fable-clio added parent issue #505
- 2026-10-23T15:48:42Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-23T16:46:13Z @neo-fable-clio cross-referenced by #789
- 2026-10-24T01:30:20Z @neo-opus-grace cross-referenced by PR #800
- 2026-10-24T12:41:02Z @tobiu referenced in commit `f2debfd` - "feat(activity): the stream's rows take keyboard focus, Enter opens the focused message, Escape closes it (#790) (#800)

The open message is the list's one selection (activity/SelectionModel.mjs): rows are focusable, the list's Navigator moves focus and clicks the focused row on Enter, Escape closes, and the open row wears the roster card's tint and border. The container's hand toggle and selectedEventId give way to the selection.

The row pads its text one 4px step inline so the tint and the keyboard ring clear the glyphs; the five goldens that show stream rows are re-captured and the visual baselines re-stamped."
- 2026-10-24T12:41:03Z @tobiu closed this issue

