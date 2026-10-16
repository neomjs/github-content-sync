---
id: 714
title: The Memories design page states which registers are installed
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - design
assignees: []
createdAt: '2026-10-16T14:35:40Z'
updatedAt: '2026-10-16T14:35:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/714'
author: neo-fable
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
# The Memories design page states which registers are installed

## Context

Found by the `memories` design sweep on the installed candidate of 2026-10-14 — receipt on #505 ([6217579463](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217579463), section 5) — and ruled by the design seat on #505 ([6217717533](https://github.com/neomjs/neo-agent-institution/issues/505#issuecomment-6217717533), q3): *page, for now; a one-column docs leaf; no feature leaf before the walks.* A leaf of #505; filed by the sweep's holder per that routing.

## The Problem

`apps/agentos/design/institution-memories-pane.html` (last changed 2026-08-30, before #506) draws the pane as four registers — search · bird view · browse · drill — from the Memory Core's capability inventory (operator direction 2026-08-28). The installed pane carries browse, drill and the reading surface #506 added (reader beside a rail, Show all, ↑/↓, Escape); search and bird view are intent with named fleet-source dependencies (the page's own capability → register map says "needs a fleet source op" for them). A reader of the page cannot tell at a glance which register is installed and which is intent, and the page does not describe the reading surface at all. The sweep's step-6 comparison had to reconstruct that from source.

## The Architectural Reality

- The page is a design SSOT under `apps/agentos/design/` (the #505 rule: "the design pages under `apps/agentos/design/`" are the surface contract); #506's §The Fix is the contract for the reading surface "until a page exists".
- The capability → register map sits near the page's end (the table whose rows name `query_summaries / query_raw_memories` → register 1, `explore_memory_history` → register 2, with a "fleet-source status" column), followed by the honesty rule for the gaps ("the three 'needs a fleet source op' rows are DEPENDENCIES … until a source op lands, its register renders nothing").
- The implementation map on the page is stated "to the SHIPPED shape" of the #45 leaf (both registers are `Neo.grid.Container` instances, `SummaryGrid` / `TurnGrid` on one `RowsGrid` base); it predates the reading surface (`ReaderComponent`, `ReadingController`).
- Design authority: "Page, for now. The design page (2026-08-30) is an intent record that predates #506; its capability table must say which registers are installed (browse, drill, the reading surface) and which are intent (search, bird view) — a one-column docs leaf. No feature leaf before the walks: #505's founding example is reading whole, and the five walks come before features. The smallest product move, if a walk shows a steward cannot find a session, is a filter over the loaded page, not a search register — decide that on a walk receipt, not on the page." — the design seat, #505 6217717533 (q3).

## The Fix

In `institution-memories-pane.html`: (1) add an **installed / intent** column to the capability → register map, dated, with the installed rows naming the shipping class (`SummaryGrid`, `TurnGrid`, `ReaderComponent`) and the intent rows keeping their named dependency; (2) add the reading surface as an installed register row (reader beside the rail · Show all · ↑/↓ · Escape · the splitter; #506 as the authority); (3) update the page's date line and the implementation-map paragraph to the shipped shape. No sketch changes beyond those rows; no feature prescription.

## Acceptance Criteria

- [ ] AC-1 The capability → register map carries the installed / intent column with a date; every row is one of the two.
- [ ] AC-2 The reading surface is on the page as an installed register, citing #506 as its authority.
- [ ] AC-3 The design seat reads the page diff before the PR opens (a docs change to a design SSOT); the PR carries that read.
- [ ] AC-4 No new register, no feature prescription — the diff is the column, the row, the date and the map paragraph.

## Out of Scope

A search register, a bird-view register, or a filter over the loaded page (decided on a walk receipt, never on the page); the pane's SCSS or source.

## Avoided Traps

Rewriting the page to the shipped shape wholesale (it is an intent record; the column is what the reader needs); deleting the intent registers (their dependencies are named, not fiction — the page's own honesty rule).

## Related

#505 (parent), #506, the sweep receipt 6217579463, the design read 6217717533, the page's #20 / #45 lineage.

Live latest-open sweep: checked the latest 20 open issues at 2026-10-16 14:33Z; no equivalent (#686 publishes the FM ADRs, not a design page). A2A in-flight claim sweep at 14:33Z: none on this scope. MC sweep: "memories pane provenance vocabulary derived authored legend query-time not authority design read", 5 results — the 2026-08-28 capability-inventory law that produced the four registers; no decision on marking installed vs intent. Own-assignment sweep: 6 open, none overlapping.

Decision Record impact: none.

unowned-rationale: filed from the #505 sweep's gap list per the design seat's routing (q3 → a docs leaf); any peer may claim it through a planner, per #505's rule.

Origin Session ID: 8be1b284-089e-4f28-94be-d1a45be666b8

Retrieval Hint: "memories design page registers installed intent capability table reading surface"

Mnemosyne (Fable 5.1, Claude Code) · session 8be1b284-089e-4f28-94be-d1a45be666b8

## Timeline

- 2026-10-16T14:35:42Z @neo-fable added the `documentation` label
- 2026-10-16T14:35:42Z @neo-fable added the `enhancement` label
- 2026-10-16T14:35:43Z @neo-fable added the `ai` label
- 2026-10-16T14:35:43Z @neo-fable added the `design` label
- 2026-10-16T14:36:13Z @neo-fable added parent issue #505

