---
id: 375
title: 'The Observatory''s team filter, a way back from a focus, a two-row head'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T22:24:05Z'
updatedAt: '2026-09-30T22:52:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/375'
author: neo-opus-grace
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
---
# The Observatory's team filter, a way back from a focus, a two-row head

## Context

@tobiu walked the Observatory on the installed Fleet Manager (2026-09-30, a 140,644-node read) and gave four notes:
1. Team lists every contributor, not just the maintainers. A team/all filter could help.
2. The list is alphabetical; it would read better by node count, descending.
3. Selecting a list item draws a green focus border, the graph loses its initial colouring, and he could not get back to the first state.
4. The head row is overloaded; two rows would read better.

The screenshot bears each one out. Outside contributors' handles lead the Team list: raw code-unit order puts `@A…` before `@neo-…`, so the maintainers and the owner sit below the list's 132 px fold. The head ends in an ellipsis where the hovered node's name pushes the read's line out.

## The Problem

- **Team is everyone.** `peersOf` counts every identity the read attributes nodes to: issue/PR authors and assignees, and memory identities. On the live corpus most of those are outside contributors.
- **No way back from a focus.** A selected node fades everything but its neighbourhood (`apps/agentos/canvas/Observatory.mjs:15`). The only exit is a click on the empty surface (spec: "a click on the empty surface clears"), and a 140k-node cloud has little empty surface to click. A second click on the selected row re-selects it, because the engine's list model toggles only in multi-select (`Neo.selection.ListModel#onListClick`, engine pin 067f9fb93b). No key clears, and no control says how. The lens does have a way back (a second click unchecks a peer), but nothing names it, and several checked peers need one click each.
- **The outline follows the mouse.** The engine outlines the navigator's active item (`.neo-navigator-active-item`, `--list-item-focus-outline`), so a mouse click leaves the signal-coloured border the SCSS comment meant for keyboard focus.
- **One row for three things.** The head is one nowrap flex row: title, the read's line, and the hovered node.

## The Architectural Reality

- **The Brain already knows the team.** Its scene read carries `AgentIdentity` nodes (wire kind string `AgentIdentity`, scene id `<origin>#@login`) for the identities it registered.
  - Live read 2026-10-01: `@neo-opus-grace` is an `AgentIdentity` labelled "Grace", and `@tobiu` is one labelled as the human owner. An outside contributor's handle has no node.
  - A small `get_graph_scene` read listed twelve agent identities first.
  - So the team is a join inside the read, with no roster in the Institution and no wire change. `peersOf`'s doc keeps its point: the Fleet registry need not know a peer.
- **The code this touches:**
  - `ObservatorySceneLayout.fromGraphScene` / `peersOf` (`apps/agentos/util/ObservatorySceneLayout.mjs:227`, `:605`).
  - `ObservatoryContainer`: `lensPeers_`, `fillPeerList`, `onPeerSelectionChange`, `onNodeSelect`, `updateLine`, `onNodeHover`, and the head vdom `[title, currency, hover]` (`apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs`).
  - `ObservatorySelectionContainer`'s actions row.
  - The shared list variables in `ObservatoryContainer.scss`.
- **Escape has a precedent:** the engine's `keys` config (`Neo.util.KeyNavigation`), as `dialog/Base` binds it: `{Escape: 'onKeyDownEscape'}`.

## The Fix

1. **Team membership in the layout.** `fromGraphScene` records `identities`: the local ids of the read's `identityKinds` nodes (`['AgentIdentity']`), taken before mail and halo filtering so a view choice never changes who is on the team. `peersOf` marks each peer `team` and orders by node count, descending, with the identity as the tie-break.
2. **A Team section with a scope.** The Team head and list move into their own container, `ObservatoryTeamContainer`, beside the View and Selected node sections. The pane keeps the lens (`lensPeers_`), and the section owns `peerScope_` (`team` | `all`, default `team`). Moving them also brings the pane back under the app-file size bar (930 → 896 lines).
   - The Team head carries an `All` toggle, plus a `Clear` verb while the lens holds a peer.
   - In team scope the list holds the team plus any checked peer, so a check is never hidden.
   - A read that holds no identity node lists everyone and says so.
3. **A way back from a focus.** The selected node's actions gain `Clear`. Escape anywhere in the pane backs out one step: the selection first, then the lens. Component keys bubble, so a focused row reaches the pane.
4. **Keyboard-only outline.** The Observatory's lists outline the active row for keyboard focus only (`:focus-visible` re-values `--list-item-focus-outline`).
5. **A two-row head.** The first row holds the title and the hovered node (or the gesture hint). The second holds the read's line, whole, which no hover truncates.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `scene.identities` from `fromGraphScene` | the read's `AgentIdentity` nodes | the local ids (`@login`) of every identity node in the read, whatever the view hides | `[]` for a read without identity nodes | `fromGraphScene` JSDoc | layout arms: mail off, halo off, multi-origin, none |
| `peersOf(scene)` rows | the read's attribution | `{id, nodes, team}`, ordered by `nodes` descending, then `id` | `team: false` everywhere when `identities` is empty | `peersOf` JSDoc | layout arms for order and flag |
| `ObservatoryTeamContainer.peerScope_` | the Team head's `All` toggle | `team` lists the team plus checked peers; `all` lists every peer; the title counts `Team · <team> of <all>` | no identity in the read: every peer listed, and the title says the read names no team | class JSDoc | container arms; visual golden |
| clearing a focus | the Selected node's `Clear`, Escape, the Team head's `Clear` | the selected node's `Clear` sets `selectedId` to `null`; the Team `Clear` empties `lensPeers`; Escape does the first, then the second | a click on the empty surface still clears | JSDoc on the handlers | container arms; a real-browser arm (Escape from a focused row) |
| the list outline | `--list-item-focus-outline` | outlined for keyboard focus only | none | SCSS comment | real-browser arm: a mouse click leaves no outline, an arrow key draws one |
| the head | `.fm-observatory-head` | two rows: title + hover, then the line | none | SCSS comment | real-browser geometry arm; visual goldens re-recorded |

## Acceptance Criteria

- [ ] AC-1: On a read that holds identity nodes, Team lists only the peers the read holds an identity for, plus any checked peer. `All` lists every attributed identity. The title reads `Team · <team> of <all>`. A read with no identity node lists everyone and says the read names no team.
- [ ] AC-2: Peers are ordered by node count, descending, with the identity ascending on a tie.
- [ ] AC-3: From a selected node the viewer returns to the unselected scene through the Selected node's `Clear` and through Escape, with no canvas click. From a lens, the Team head's `Clear` unchecks every peer in one click. Escape backs out one step: the selection first, then the lens.
- [ ] AC-4: In a real browser, a mouse click on a list row draws no outline, and an arrow key does.
- [ ] AC-5: In a real browser, the head reads in two rows, and a hovered node never truncates the read's line.
- [ ] AC-6 (post-merge, operator): on the installed FM, Team opens on the maintainers busiest first, a selection and a lens each clear in one step, and the head reads in two rows.

## Out of Scope

- A second click on the selected node's row clearing it. The engine's single-select model does not toggle, and an engine change is a separate ticket if the Clear action and Escape prove not enough.
- Social names in the Team rows. The identity nodes carry them, which is a follow-up if wanted.
- Any Brain wire change.

## Avoided Traps

- **A roster in the Institution** (a list of maintainer handles). It would name one tenant, and an outside operator's team is not ours.
- **The Fleet roster as the team.** It holds this machine's residents, not the institution.
- **Deriving the team from the drawn scene.** A hidden halo or mail would change who is on the team.

## Related

Epic #312; team lens #320; panel #333; node list #258.

Decision Record impact: none.

Live latest-open sweep: checked the latest 20 open Institution issues at 2026-09-30T22:22Z; no equivalent found.
MC sweep: "Observatory team lens lists every identity…, outside contributors…" and "Observatory selection fades the scene, click on empty surface clears, cannot restore…", 11 results, no prior decision found. The operator's 09-24 PoC ask named the team (contributors), and this refines it.
Own-assignment sweep: 2 open (#370, #11), none overlapping.
A2A claim sweep: the last 30 inbox messages; no Observatory claim.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636
Retrieval Hint: "Observatory team scope identity nodes clear selection two-row head"



## Timeline

- 2026-09-30T22:24:06Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T22:24:06Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T22:24:06Z @neo-opus-grace added the `agent-os` label
- 2026-09-30T22:24:07Z @neo-opus-grace added the `ai` label
- 2026-09-30T22:24:07Z @neo-opus-grace added the `design` label
- 2026-09-30T22:24:09Z @neo-opus-grace added parent issue #312
- 2026-09-30T22:57:02Z @neo-opus-grace cross-referenced by PR #377
- 2026-09-30T23:45:44Z @neo-opus-grace referenced in commit `733e762` - "fix(observatory): an outsider leaves Team with its check, and a member's check keeps the rows (#375)"

