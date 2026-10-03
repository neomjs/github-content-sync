---
id: 506
title: 'Memories read in full: a reading pane for summaries and session turns'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-03T11:58:53Z'
updatedAt: '2026-10-03T12:24:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/506'
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
# Memories read in full: a reading pane for summaries and session turns

## Context

Operator, on the installed Fleet Manager, 2026-10-03: the Memories view's session-summary cards are so small that only a truncated fraction is readable, with no *expand* or *show all*; drilling into a session's memories repeats the pattern — a truncated preview, no expand. *"The combination of both failures makes the entire memories visualization useless."* First leaf of epic #505 (every important view reachable, roomy, correct, readable); the view the operator named first.

## The Problem

The Memories pane (`apps/agentos/view/fleet/memories/Container.mjs`, `SummaryGrid` → `TurnGrid` by `drillSession`) is a list of previews with no reading surface: `SummaryRowComponent` clamps its text to two lines (`Container.scss` `-webkit-line-clamp: 2`), `TurnRowComponent` cuts prose at a presentation bound (`max=600`, ellipsis) and clamps the rest to one or two lines, and nothing opens the whole. The pane shares the lower dock with Activity, Tasks, Mailbox, Catch up and Golden Path, so the list has neither height nor width to read in. Choosing whose memories to read is already an explicit act (select a card); reading them is impossible.

## The Architectural Reality

- Data: summaries and turns arrive through the admitted plane client (summaries by `get_all_summaries` / the cockpit's projection; turns by the recency read); `TurnRowComponent` notes the bounded response head "stands in meanwhile" — the wire may carry a head, not the whole turn. The reading pane needs the whole record: if the current projection truncates, the leaf adds the one read that returns a full record (`get_session_memories` / the turn by id) — a producer half in the Brain, filed as its own leaf by a planner only if the read does not exist.
- Room: the pane is a dock item; the engine's dock already offers `setActiveItem`, `moveItem`, `resizeEdgeZone`, `setItemAutoHidden` and pop-out — a reading mode is a dock operation (the pane takes the main area or pops out), not a modal.
- Surface contract: no CARD-CONTRACT row owns this pane; the design pages under `apps/agentos/design/` predate the drill-in. This leaf's §The Fix is the contract until a page exists.

## The Fix (design decision)

1. **A reading pane inside the Memories view.** Selecting a summary opens it in a reading pane beside (wide) or below (narrow) the list: title, session, agent, time, and the FULL summary text, with the drill into the session's turns one click away. Selecting a turn opens the full turn: prompt, thought, response as three readable blocks with a copy action each, the turn's metadata (time, tool-call count, authoring identity) in one line. ↑/↓ moves the selection; Esc returns to the list.
2. **No clamp without an affordance.** The list rows keep their two-line previews (the scanning density is right), but every clamped row is the affordance: click to read in the pane; a *show all* toggle on the list expands the previews in place to their full text for scanning. A `title` attribute is never the only way to the whole.
3. **Room, by default first.** The Memories view must read well where the default perspective puts it (today: one of six tabs in the lower strip — see #505's default-perspective leaf, which decides its home); within that home the reading pane collapses the list to a rail of titles so the text gets the width. Room beyond the default comes from the engine, not from the pane: the dock's Maximize (`plugin.Maximize`, on by default in `Neo.dashboard.dock.Workspace`; presentation only — Escape restores, saved perspectives untouched) gives the Memories node the workspace and the reading pane uses the width; pop-out stays. No `moveItem` round-trip: that would rewrite the operator's saved layout. Both are bonuses on top of a readable default, never the fix.
4. **Full records.** The reading pane renders the whole record from the plane; if the current projection carries only a head, the pane reads the full record by id on open (one read per open, cached per record) — never the `max=600` cut.
5. **Honest states, existing words.** Nothing selected → the pane says "select a summary to read it"; a record the plane did not return in full → the pane says so with the id, never a silent head.

## Acceptance Criteria

- [ ] AC-1 Unit arm: selecting a summary renders the reading pane with the full summary text (a 3,000-character fixture reads whole, no ellipsis); selecting a turn renders prompt/thought/response in full with three copy actions; ↑/↓/Esc behave as specified.
- [ ] AC-2 Unit arm: *show all* expands every row's preview in place and back; a clamped row is clickable and opens the pane.
- [ ] AC-3 NL/e2e arm: the dock's Maximize gives the Memories node the workspace and the reading pane uses the width; Escape restores the layout and the saved perspective is unchanged; pop-out still works. (If Maximize does not project on `stream-tabs`, the capture says so and the planner decides.)
- [ ] AC-4 The full record read: a turn whose wire row carries a bounded head reads whole in the pane (fixture with a head shorter than the record); the unavailable case renders the honest sentence with the id.
- [ ] AC-5 Design read on the installed candidate with the team's data BEFORE the PR opens (a capture of the list + reading pane at the lower-dock size and in the main area), approved by the design seat; goldens re-captured from a full visual run.
- [ ] AC-6 (post-merge, installed) On the next #12 cut the operator opens one of his own session summaries and one turn and reads both in full; one screenshot receipt on this ticket.

## Out of Scope

- Editing or deleting memories from the cockpit.
- Semantic search across memories (a later leaf of #505 if the operator wants it).
- The Thought stream pane of the Agent Detail (#476) — it reads turns too, but it is a status pane, not the reading surface.

## Avoided Traps

- A modal dialog for reading: the dock already has the room operations; a modal fights the layout the operator arranged.
- Raising the clamp bound (`max=600` → more): more preview is still a preview; the affordance to the whole is the fix.
- Loading every full record into the list: the list stays light; the reading pane fetches one whole record on open.

## Related

#505 (epic — parent), #476 / Brain #792 (turn summaries on the wire), the Memories pane's "explicit selection" rule (Container.mjs), the engine's dock operations (used by the cockpit's perspectives), #12 (the installed cut that carries AC-6).

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-03 11:56Z; no equivalent. A2A in-flight claim sweep at 11:56Z: none on the Memories view. Memory Core rationale sweep: the pane's prior decision is "choosing whose memories to read is an explicit act" — kept; no prior decision on reading affordances. Own-assignment sweep: no open ticket of mine owns the Memories view. Structure map: N/A — Institution view layer; owning folder `apps/agentos/view/fleet/memories/`.

unowned-rationale: filed by the planner under the filing freeze; the first free peer claims it through a planner (the design read in AC-5 is the planner's gate); the Brain producer half, if needed, is filed by a planner once the claimer reports the wire shape.

Retrieval Hint: "memories reading pane summaries turns full record show all dock maximize no clamp without affordance"

Amended 2026-10-03 12:2xZ on Grace's fork: room beyond the default is the engine's Maximize, not a moveItem; Fix 4 struck with proof (listMemories returns prompt/thought/response uncut at fb40366).

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb



## Timeline

- 2026-10-03T11:58:54Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T11:58:54Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T11:58:54Z @neo-fable-clio added the `ai` label
- 2026-10-03T11:58:55Z @neo-fable-clio added the `design` label
- 2026-10-03T11:59:09Z @neo-fable-clio added parent issue #505
- 2026-10-03T12:04:34Z @neo-fable-clio cross-referenced by #507
- 2026-10-03T12:11:34Z @neo-fable-clio assigned to @neo-opus-grace
- 2026-10-03T12:21:39Z @neo-opus-grace cross-referenced by #505

