---
id: 626
title: 'The Observatory''s Nodes list leads with what changed, and names its order'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T04:44:58Z'
updatedAt: '2026-10-09T15:17:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/626'
author: neo-opus-vega
commentsCount: 0
parentIssue: 312
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-09T15:17:40Z'
milestone: FM v1
---
# The Observatory's Nodes list leads with what changed, and names its order

Sub of #312 (row 3 of FM v1). One of three leaves from the row-3 walk's gap list, design read passed; the window decision is shared with #625.

## Context

Row 3's warm walk (2026-10-03, #485 comment 5971599390, step 6) found a May Discussion with one relation at rank 2 of the Nodes list, the first thing a stranger reads in the side panel. The designated reader ruled (#485 comment 5971636965, step 6):

> The **default order is attention** — nodes whose state changed since the last capture first (a row that failed, a claim that landed, a PR that merged), then by activity; and the **ranking criterion is named in the list header** (`ranked by: changed since 06:20 PM`), with the current degree-style order available as a second named sort. "The data exists" is never a reason for its position.

Verified on Institution `dev` 32627ab: `ObservatoryContainer.fillNodeList()` lists `scene.nodes.slice(0, listBudget)` in the layout's order, the route's seeds first; the section title reads `Nodes` or `Nodes · <n> of <m> · relations reach the rest`, and names no order.

## The Problem

The list's top is where a stranger's attention goes, and today it is filled by the layout's order, which encodes geometry, not news. A merged PR or a moved ticket from this morning can sit below the fold of a 1,981-row list.

## The Architectural Reality

- Each node carries `kind`, `state`, `lastActivityAt` and `label` (Brain #603). `ObservatorySceneLayout.heatEvents` already names which kinds carry attention (`ISSUE`, `PULL_REQUEST`, `DISCUSSION`, `AGENT_MEMORY`, `KB_GAP`, `RETROSPECTIVE`, `TOOLING_GAP`) and which change without it (`MESSAGE`, `FILE`, `DIRECTORY`).
- A merged PR heats 0 (`heatOf()` retires it), yet the ruling puts it first: the list's criterion is *changed*, not *hot*.
- The list projects at most `listBudget` rows, so the order must apply to the whole read before the budget cuts it. The selected node outside the budget stays added as one extra row.
- `ObservatoryContainer.mjs` is 996 lines against the 1,000-line bar: the ordering lives in a static over the scene (`ObservatoryNodeList.orderOf`, beside the list that reads it), and the container only asks for it.
- Design authority: the reader's ruling above. The layout order was never chosen for reading; it is the order the scene arrives in.

## The Fix

1. Default order: nodes of an attention-bearing kind whose `lastActivityAt` falls within the window, most recent first, merged and closed items included; then every other node by its last activity, newest first. The window is #625's decision: the heat's stated 3 days.
2. The section title names the order: `Nodes · changed in the last 3 days first · then by activity`, keeping the `<n> of <m>` budget words.
3. A second named sort, `by relations`: the most related first, ties in today's order (the route's seeds, then by id). It sits in the section's head; the choice is session view state.
4. Unit coverage on a fixture scene (merged PR, moved issue, old discussion, message, file); the NL e2e arms that read the list keep passing.

Items 1–3 follow the design seat's decision on #625 ([comment 6074465175](https://github.com/neomjs/neo-agent-institution/issues/625#issuecomment-6074465175)), which added `then by activity` and named the second sort `by relations`. Today's order is the route's seeds, then every node by id, which is not a relation order, so the name decides the sort.

## Acceptance Criteria

- [ ] AC-1: by default, the list's first rows are the attention-bearing nodes changed within the window, newest first, a merged PR among them; messages and files never lead on a change alone (unit).
- [ ] AC-2: the order applies before the list budget: a changed node past the layout's first `listBudget` nodes still lists (unit).
- [ ] AC-3: the title names the order, and `by relations` reads the most related first (unit; NL e2e on the live-shaped read).
- [ ] AC-4: `ObservatoryContainer.mjs` stays under the bar; the panel goldens are re-captured and `check-visual-baselines` is green.

## Out of Scope

The head's first line (#625). Names (#624). New reads (ROADMAP rows, claims, reviews).

## Related

Parent #312 · walk #485 (comments 5971599390, 5971636965) · #624 · #625 · Brain #603.

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-09T04:42:40Z; no equivalent.
A2A claim sweep: last 30 messages (all read-states) at 04:44Z; no claim on the Observatory.
MC sweep: "… Nodes list ordered by attention changed since capture", 8 results: the reader's 2026-10-03 ruling, accepted for the gap list and never filed; no other decision.
Own-assignment sweep: 2 open (#599, #485), none overlapping.

unowned-rationale: Vega builds #624 then #625 tonight; this leaf is free for any peer who can start it sooner, otherwise Vega takes it after #625.

Origin Session ID: 9a84c569-02eb-4f7c-b87d-43ebcb24593d
Retrieval Hint: "Observatory Nodes list attention order changed within window ranked by route"

## Timeline

- 2026-10-09T04:45:00Z @neo-opus-vega added the `enhancement` label
- 2026-10-09T04:45:00Z @neo-opus-vega added the `agent-os` label
- 2026-10-09T04:45:00Z @neo-opus-vega added the `ai` label
- 2026-10-09T04:45:00Z @neo-opus-vega added the `design` label
- 2026-10-09T04:45:08Z @neo-opus-vega added parent issue #312
- 2026-10-09T04:45:17Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-10-09T04:45:49Z @neo-opus-vega cross-referenced by #312
- 2026-10-09T04:47:41Z @neo-fable-clio cross-referenced by #625
- 2026-10-09T05:31:15Z @neo-opus-vega cross-referenced by PR #630
- 2026-10-09T05:37:34Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T06:02:38Z @neo-opus-vega cross-referenced by PR #631
- 2026-10-09T06:34:55Z @neo-opus-vega referenced in commit `b4f1c3f` - "feat(observatory): the Nodes list leads with what changed, and names its order (#626)

The list orders the whole read before the budget cuts it: work that changed within the
attention window first, newest first, merged and closed included, then every node by its
last activity. A message or a file never leads on a change alone. The head names the order
in the design seat's words and wraps beside its toggle, which reads by relations and back.
factsOf moves to the selection section it feeds, keeping the container under the bar."
- 2026-10-09T06:34:56Z @neo-opus-vega referenced in commit `8280df3` - "feat(observatory): the Nodes head names its order on a line of its own, in the detail role (#626)

The design read moved the order off the chrome line: `Nodes · 2 of 7` keeps the section's
word and the budget beside the toggle, and the order sits beneath in dim detail text, `by
relations · most related first` when pressed. A collapsed section keeps only its chrome line.
The head moves into ObservatoryNodesHeadContainer, which owns the words and both controls;
the container drops from 996 to 973 lines and keeps one item and two event hooks."
- 2026-10-09T06:50:09Z @neo-opus-vega cross-referenced by #485
- 2026-10-09T12:19:04Z @neo-opus-vega referenced in commit `fc669f0` - "chore(observatory): merge the synced head's sentence into the Nodes order, the visual stamp regenerated (#626)"
- 2026-10-09T13:15:59Z @neo-opus-vega referenced in commit `8511de5` - "chore(observatory): merge the head's sentence synced after #629 into the Nodes order, the visual stamp regenerated (#626)"
- 2026-10-09T14:39:36Z @neo-opus-vega referenced in commit `2632158` - "chore(observatory): merge dev after #641 into the Nodes order, the visual stamp regenerated (#626)

Only the visual stamp conflicted. Regenerated after a full visual run (49/49); units 1633 passed, 11 skipped."
- 2026-10-09T14:45:50Z @neo-opus-vega referenced in commit `8a76ea2` - "chore(observatory): merge dev after #628's squash into the Nodes order, conflicts resolved to the branch's side (#626)

#631 is stacked on #628 (through 7271700) and #630 (dd0096d), so it already held #628's final content; the squash 4204bb8 has the same tree as #628's head 2661510. Every conflict resolved to this branch's side, and the merged tree is byte-identical to 2632158, which passed visual 49/49 and units 1633. The stamp still matches."
- 2026-10-09T15:06:56Z @neo-opus-vega referenced in commit `bebe548` - "chore(observatory): merge dev after #630's squash into the Nodes order, conflicts resolved to the branch's side (#626)

#631 already held all of #630: every #630 commit is an ancestor, and the squash 29ab1b3 has the same tree as #630's head 001d8d1. All 18 conflicts resolved to this branch's side. The merged tree is byte-identical to 8a76ea2 (and 2632158), which passed visual 49/49 and units 1633; the stamp still matches."
- 2026-10-09T15:17:40Z @tobiu referenced in commit `7157391` - "feat(observatory): the Nodes list leads with what changed, and names its order (#626) (#631)

* feat(observatory): the Observatory names each surface once: its view, its Route, its pane (#624)

The head title names the view and the geography the scene drew
(`Observatory · Roadmap wells` / `Hub wells`), the View section's route control
reads `Route`, and "Golden Path" stays the lower-dock pane's name, as the row-3
reader ruled. The fourteen Observatory goldens are re-captured: every shot
frames the head, and the pixel-ratio tolerance had let the old names pass.

* test(observatory): re-capture the widened side-panel goldens, which still showed the old names (#624)

* feat(observatory): the head opens with the team's sentence, its counts behind Details (#625)

The Observatory head now leads with what the team did in the attention
window: `captured 06:20 PM · last 3 days: 4 merged · 12 in motion ·
attention: #N · <title> · complete`, with zero classes silent and the
attention item a link that selects its node. The renderer's counts, a
withheld route and the overlays' words sit behind a Details disclosure,
which shows only when they say more than the line. A pressed Roadmap on a
read without roadmap anchors reads `Roadmap · no anchors`.

The head becomes its own component (ObservatoryHeadComponent, with its own
sheet) and the sentence a util (ObservatoryBrief); the container drops from
998 to 990 lines. GraphSceneEnvelope.completenessOf and
GraphNodeSource.numberOf are shared steps the sentence reuses.

* fix(observatory): the head's sentence counts unknown work instead of a zero, and the browser opens Details (#625)

A time on one node said nothing about another's, yet any timestamp in the read made untimed
work read as "nothing moved". Work a count could hold whose own time or state the read omits
is now unknown, in the heat line's word: "· 2 unknown". The NL journey reads the visible line,
opens Details with a real click, and selects the attention item's node through the browser.

* feat(observatory): the Nodes list leads with what changed, and names its order (#626)

The list orders the whole read before the budget cuts it: work that changed within the
attention window first, newest first, merged and closed included, then every node by its
last activity. A message or a file never leads on a change alone. The head names the order
in the design seat's words and wraps beside its toggle, which reads by relations and back.
factsOf moves to the selection section it feeds, keeping the container under the bar.

* feat(observatory): the Nodes head names its order on a line of its own, in the detail role (#626)

The design read moved the order off the chrome line: `Nodes · 2 of 7` keeps the section's
word and the budget beside the toggle, and the order sits beneath in dim detail text, `by
relations · most related first` when pressed. A collapsed section keeps only its chrome line.
The head moves into ObservatoryNodesHeadContainer, which owns the words and both controls;
the container drops from 996 to 973 lines and keeps one item and two event hooks."
- 2026-10-09T15:17:40Z @tobiu closed this issue

