---
id: 636
title: 'Default perspective design page: the inventory''s homes, measured'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-10-09T06:43:48Z'
updatedAt: '2026-10-09T12:12:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/636'
author: neo-fable-clio
commentsCount: 0
parentIssue: 507
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T12:12:28Z'
---
# Default perspective design page: the inventory's homes, measured

## Context

#507 (the default perspective gives each important view a good home) opens with AC-1: a design page the operator approves before any catalog change. The approval is his (Tier 4, the default is his product opinion), so the page lands first and the catalog change waits on his record. A pull request resolves one ticket, and #507's four remaining ACs are a builder's lane after the approval — the page gets its own leaf, as the setup card's contract did (#421) beside its build (#384).

The measure is the operator's capture of 2026-10-09 02:58Z, the first morning all eight active peers ran from the Fleet Manager, as the 2000 × 1217 px copy pasted into this seat's session (a resize of the original; proportions intact, aspect 1.64; the window's logical size is not read) — the roster in three columns holding nine of twelve cards, one six-tab strip under it with eight Activity rows, Agent Detail in the right band, where a card click commits its reveal as a docked 0.25 band and only the rail tab shows a temporary overlay.

## The Problem

Today's Overview (`apps/agentos/util/CockpitPerspectives.mjs`, `arrangement()`) has an opinion about the roster and none about the rest: everything the operator reads or acts on shares ≈ 442 capture px under one tab strip, while the view he reaches most after the roster, an agent's detail, holds a band of its own that the reading views cannot share. #505's four questions (one move, room by default, renders, read in full) have no answer as a set.

## The Architectural Reality

- The dock model is the vocabulary: zones, tabs nodes with `activeItemId` (`WorkspaceDocument.mjs`, the tabs field set; an absent one is repaired to the first item), splits with `sizes`, edge bands with `extent`. A new default is a catalog change plus split numbers, not a mechanism.
- A center member keeps its catalog `autoHidden` flag and stays in the tab flow; `revealsInspector` reads placement, not the flag (Review already relies on it).
- `apps/agentos/design/` holds spec documents and is carved out of the visual-baseline stamp (`checkVisualBaselines.mjs`, the scope's exclude), so a page lands without a re-stamp.

## The Fix

`apps/agentos/design/default-perspective.html`, in the sibling pages' skeleton (the FM tokens, the five type roles): §01 the window measured on the capture; §02 today's Overview and the re-declared one drawn to the capture's proportions; §03 the numbers at the operator's window; §04 the inventory placed by daily weight with the four questions answered per view; §05 the `zones` declaration the catalog would carry; §06 what the page does not verify, and the approval slot.

The proposal the page carries, for the operator's word: the dock's center becomes a 0.64 : 0.36 horizontal split, the fleet-over-feeds stack on the left and a full-height reading column on the right; the two feeds (Activity, Catch up) stay under the roster; Agent Detail, Memories, Mailbox, Tasks and Golden Path move into the column, Memories open at first run, Detail activated by selection (it leaves its 0.25 band for the wider column it shares with the reading tabs: selecting a peer while reading takes the column to Detail, the reading tab one click back); the right band keeps the three invoked tools. Reading height ≈ 2.6× today's; the Activity feed keeps 7–8 rows; the roster trades 3 × 3 for 2 × 3 with the rest on scroll.

Contract Ledger: not applicable — a spec document introduces no surface consumed by code, agents or external systems; the catalog change (#507 AC-2) carries its own.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 The page exists on dev with the six sections above, every number measured on the capture or read in the engine's dock model, both frames rendering without overflow at 1240 px wide.
- [ ] AC-2 No source, token, test or golden change; the baseline stamp is untouched.
- [ ] AC-3 (post-merge, operator-owned) #507's AC-1 points at the page; the operator records his approval, or the changes he wants, on #507 — never here.

## Out of Scope

The catalog change, the perspective-survival arm, the goldens and the installed receipt: #507 AC-2 … AC-5, on a builder's branch after the approval.

## Avoided Traps

- Screenshots in the repository: the frames are drawn to the capture's proportions; the capture itself stays the operator's.
- Deciding the default by the dev server: the measure is the installed app's capture with the team's own data.
- Splitting the lower zone side by side (the ticket's starting direction): width, where the reading views need height.

## Related

#507 (parent leaf, holds the approval), #505 (the epic and its inventory), #632 (the retained Detail follows selection — independent of this page), #421 / #422 (the precedent: a design contract as its own leaf).

Live latest-open sweep: checked the latest 20 open issues, created-descending, at 2026-10-09 06:5xZ; no equivalent (#507 is the parent). A2A in-flight claim sweep: none on this page (the only Fleet Manager lane claim since 06:00Z is #632). Memory Core rationale sweep: no prior decision on the default perspective beyond #507.

Origin Session ID: fc9a1ad6-b0f5-47d6-99dd-3d42784c4fcb

Retrieval Hint: "default perspective design page reading column operator's capture 2026-10-09"

## Timeline

- 2026-10-09T06:43:48Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-09T06:43:50Z @neo-fable-clio added the `enhancement` label
- 2026-10-09T06:43:50Z @neo-fable-clio added the `agent-os` label
- 2026-10-09T06:43:50Z @neo-fable-clio added the `ai` label
- 2026-10-09T06:43:51Z @neo-fable-clio added the `design` label
- 2026-10-09T06:44:02Z @neo-fable-clio added parent issue #507
- 2026-10-09T06:44:41Z @neo-fable-clio cross-referenced by PR #637
- 2026-10-09T06:45:41Z @neo-fable-clio cross-referenced by #507
- 2026-10-09T07:06:53Z @neo-fable-clio referenced in commit `b610afb` - "docs(agentos): the comparison's facts: the committed reveal, the capture's scale, the leaf's ACs (#636)"
- 2026-10-09T12:12:28Z @tobiu referenced in commit `828b691` - "docs(agentos): the default perspective's design page, each important view's home at the operator's window (#636) (#637)

* docs(agentos): the default perspective's design page, each important view's home at the operator's window (#636)

* docs(agentos): the comparison's facts: the committed reveal, the capture's scale, the leaf's ACs (#636)"
- 2026-10-09T12:12:29Z @tobiu closed this issue

