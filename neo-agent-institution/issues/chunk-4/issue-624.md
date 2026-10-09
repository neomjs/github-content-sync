---
id: 624
title: 'The Observatory names each surface once: its view, its Route, its pane'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T04:43:51Z'
updatedAt: '2026-10-09T14:42:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/624'
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
closedAt: '2026-10-09T14:42:28Z'
milestone: FM v1
---
# The Observatory names each surface once: its view, its Route, its pane

Sub of #312 (row 3 of FM v1). One of three leaves from the row-3 walk's gap list, design read passed.

## Context

Row 3's warm walk (2026-10-03, #485 comment 5971599390, step 4) found three surfaces reading "Golden Path": the Observatory's head title, its View section's route control, and the lower-dock pane. The designated reader ruled on it (#485 comment 5971636965, step 4):

> One name, one thing: the **lower-dock pane** is the Golden Path (the recommendation, read in full) and keeps the name. The overlay toggle on the graph is named by what it does — **Route** (the path drawn through the wells) — not by the pane it illustrates. The head title names the view and its lens — **Observatory · Roadmap wells** — never a tab that lives elsewhere.

Verified on Institution `dev` 32627ab: `view/fleet/goldenpath/ObservatoryContainer.mjs:147` renders the title `Golden Path · observatory`; `ObservatoryViewContainer.mjs:86` labels the route control `Golden Path`.

## The Problem

A stranger who reads the same words in three places assumes three things. Here there is one thing, the Golden Path read, and two illustrations of it: a route drawn on the graph, and a view that is not a tab of the cockpit at all.

## The Architectural Reality

- The head title is a static span (`fm-observatory-title`) in `ObservatoryContainer`'s head. The view draws one of two well geographies, `strategic` (the View section's `Roadmap` button) and `density` (`Hubs`), held in its `geography_` config.
- The route control is a Button in `ObservatoryViewContainer`, synced by `ObservatoryContainer.updateView()`; its class doc reads "Golden Path draws the route".
- `ObservatoryContainer.mjs` is 996 lines against the 1,000-line app-file bar: the title must not grow it (derive the words where the geography is named, or move a block out).
- Design authority: the reader's ruling above. The current title has no record making it intended; the control's name came with the route overlay (#310).

## The Fix

1. The head title reads `Observatory · Roadmap wells` under the strategic geography and `Observatory · Hub wells` under the density one, and follows a geography switch.
2. The View section's route control reads `Route`; its tooltip, aria name and class doc follow. The withheld-route line keeps naming the Golden Path read as its source (`route withheld · …`), and the selected node's `Golden Path rank N` fact stays, because it names the pane's ranking.
3. The lower-dock pane keeps "Golden Path".
4. Unit and NL e2e locators that drive the control by its old name move to `Route`; the Observatory goldens are re-captured.

## Acceptance Criteria

- [ ] AC-1: the head title reads `Observatory · Roadmap wells` / `Observatory · Hub wells` by the drawn geography, and follows a switch (unit).
- [ ] AC-2: the View section's route control is named `Route` in its text, tooltip and accessible name (unit; the NL e2e arms that press it).
- [ ] AC-3: in the Observatory, "Golden Path" appears only where it names the Golden Path read or its ranking, never as a surface's name (unit over the head and the View section).
- [ ] AC-4: `ObservatoryContainer.mjs` stays under the 1,000-line bar (`check-app-file-sizes`); the touched goldens are re-captured and `check-visual-baselines` is green.

## Out of Scope

The head's first line (its sibling leaf: the team's sentence first). The Nodes list's order (its sibling leaf). Renaming the lower-dock pane.

## Related

Parent #312 · walk #485 (comments 5971599390, 5971636965) · #310 (the route overlay) · #510 (the Golden Path pane read in full).

Live latest-open sweep: latest 20 open Institution issues read at 2026-10-09T04:42:40Z; no equivalent.
A2A claim sweep: last 30 messages (all read-states) at 04:44Z; no claim on the Observatory.
MC sweep: "Observatory head first line team's sentence … one name per surface Route overlay …", 8 results: the reader's 2026-10-03 ruling, accepted for the gap list and never filed; no other decision.
Own-assignment sweep: 2 open (#599, #485), none overlapping; #485's body names no naming leaf.

Origin Session ID: 9a84c569-02eb-4f7c-b87d-43ebcb24593d
Retrieval Hint: "Observatory one name per surface Route control head title Roadmap wells"

## Timeline

- 2026-10-09T04:43:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T04:43:53Z @neo-opus-vega added the `enhancement` label
- 2026-10-09T04:43:53Z @neo-opus-vega added the `agent-os` label
- 2026-10-09T04:43:54Z @neo-opus-vega added the `ai` label
- 2026-10-09T04:43:54Z @neo-opus-vega added the `design` label
- 2026-10-09T04:44:30Z @neo-opus-vega cross-referenced by #625
- 2026-10-09T04:44:59Z @neo-opus-vega cross-referenced by #626
- 2026-10-09T04:45:04Z @neo-opus-vega added parent issue #312
- 2026-10-09T04:45:14Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-10-09T04:45:49Z @neo-opus-vega cross-referenced by #312
- 2026-10-09T04:54:01Z @neo-opus-vega cross-referenced by PR #628
- 2026-10-09T05:27:21Z @neo-opus-vega referenced in commit `84dbc9f` - "test(observatory): re-capture the widened side-panel goldens, which still showed the old names (#624)"
- 2026-10-09T05:46:32Z @neo-gpt-emmy cross-referenced by PR #630
- 2026-10-09T06:50:09Z @neo-opus-vega cross-referenced by #485
- 2026-10-09T12:12:22Z @neo-opus-vega referenced in commit `f29e44f` - "chore(observatory): merge dev into the Observatory names, the visual stamp regenerated over both sides (#624)"
- 2026-10-09T13:08:13Z @neo-opus-vega referenced in commit `7271700` - "chore(observatory): merge dev after #629 into the Observatory names, the visual stamp regenerated (#624)"
- 2026-10-09T14:14:55Z @neo-opus-vega referenced in commit `2661510` - "chore(observatory): merge dev after #641 into the Observatory names, the visual stamp regenerated (#624)

Only the visual stamp conflicted. Regenerated from the merged tree after a full visual run (49/49 passed); units 1621 passed, 11 skipped."
- 2026-10-09T14:42:28Z @tobiu referenced in commit `4204bb8` - "feat(observatory): the Observatory names each surface once: its view, its Route, its pane (#624) (#628)

* feat(observatory): the Observatory names each surface once: its view, its Route, its pane (#624)

The head title names the view and the geography the scene drew
(`Observatory · Roadmap wells` / `Hub wells`), the View section's route control
reads `Route`, and "Golden Path" stays the lower-dock pane's name, as the row-3
reader ruled. The fourteen Observatory goldens are re-captured: every shot
frames the head, and the pixel-ratio tolerance had let the old names pass.

* test(observatory): re-capture the widened side-panel goldens, which still showed the old names (#624)"
- 2026-10-09T14:42:29Z @tobiu closed this issue
- 2026-10-09T15:06:04Z @tobiu referenced in commit `29ab1b3` - "feat(observatory): the head opens with the team's sentence, its counts behind Details (#625) (#630)

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
opens Details with a real click, and selects the attention item's node through the browser."
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

