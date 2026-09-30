---
id: 333
title: 'The Observatory''s panel: View, then the selected node and its source'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-29T20:35:32Z'
updatedAt: '2026-09-29T23:44:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/333'
author: neo-opus-vega
commentsCount: 1
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
closedAt: '2026-09-29T22:42:27Z'
---
# The Observatory's panel: View, then the selected node and its source

## Context

D#19317 §4 is the Observatory's right-hand panel contract: Team at the top (#320 shipped it), then View (Q1–Q3), then the selected node (Q5). #312 has not filed the last two yet. Emmy's epic review names Q5 as required before the epic can close ([closeout matrix](https://github.com/neomjs/neo-agent-institution/issues/312#issuecomment-5869511213)). This leaf adopts Euclid's installed witness and acceptance proposal on #312 ([comment 5870358648](https://github.com/neomjs/neo-agent-institution/issues/312#issuecomment-5870358648)) as its boundary.

## The Problem

- **The selection is one line of the strip:** `Selected · label · kind · qualified id · rank · N relations (…)` (`ObservatoryContainer#describeSelection`), with a flat relation list under the node list.
  - The scene carries each node's state, attribution and last activity (B1–B3), but the selection shows none of them.
  - The raw id sits in the headline.
  - Nothing opens the node's source.
- **The View controls are named by mechanism:** Mail, Halo, Heat and Route sit in the strip above the canvas. The geography (`strategic`, `density`) is a config with no control.
- **Where the evidence lives,** measured on the scene read at Brain dev@`83c0e09` on 2026-09-29 at 19:38Z (152,675 nodes):
  - `get_node`, sampled once per kind, returned an empty description for an issue, a PR, a discussion, a memory and a message. The two gap nodes sampled read "None." and "tag: N/A — none added." An inline summary would be empty or invented. The evidence is at the source.
  - 19,154 nodes (12.5%) are canonical GitHub work items, meaning the kind and the id (`origin#issue-N`, `#pr-N`, `#discussion-N`) agree: every PR and discussion, and 12,329 of the 18,361 issues. Another 23 nodes carry such an id under another kind (CONCEPT 11, CLASS 8, ARTIFACT_TASK 4).
  - 1,737 of the 7,087 SESSION nodes have a `session:<uuid>` id, and 322 of the 326 summaries a `summary_<uuid>` id. Both uuids are session ids that `get_session_memories` reads (checked for one of each).
  - 35,950 of the 43,589 FILE nodes name paths under `.venv/` (9,981) or `resources/content/` (25,969). Neither is in the engine's dev tree, so a link would 404.

## The Architectural Reality

- **The container:** `ObservatoryContainer.mjs` has 949 lines and three regions: the head; the strip (the selection line and four toggles); and the body (the side panel's Team, Nodes and Relations lists, which the canvas joins). `selectedId_` binds two-way to the Viewport's `graphSelectionId`.
  - A selection only inks the scene (`ObservatoryCanvas#afterSetSelectedId`), so the camera stays where it is.
  - `ObservatoryCanvas#readStats` reports the camera, which an NL arm can read.
- **The file-size bar:** `buildScripts/checkAppFileSizes.mjs` warns at 900 lines and fails above 1,000, and the container is in the warning band.
- **Grouped rows:** `tasks/List.mjs` consumes the engine list's header records (`useHeaders`, `isHeader`).
- **GitHub links:** `catchup/Container.mjs:23` and `:450` render an anchor with `target: '_blank'` and `rel: 'noopener noreferrer'`.
- **The session drill** belongs to the cockpit: `cockpit/Controller.mjs#loadSessionMemories({sessionId, title})`, fed by the Memories pane's `sessionDetailRequest`. The drill's read is viewer-scoped. The Observatory is a rail view (`/observatory`), so opening a session takes three steps: a route change, activating the Memories tab, and that seam.
- **Activating a resident tab:** `ReadingSurfacesController#openCatchUpLiveSurface` does it for the Activity stream, through `dockModel.nodes[…].items` and the tab strip's `activeIndex`.
- **Vocabularies:** `ObservatorySceneLayout` already declares its vocabularies as configs (`heatEvents`, `workStates`). `KindRegistry` covers fleet event kinds, not graph node kinds.

## The Fix

1. **The panel:** it reads Team, View, Selected node, per §4. The strip leaves and the canvas takes its row. The sections become components beside `ObservatoryPeerList`, `ObservatoryNodeList` and `ObservatoryRelationList`, so the container shrinks rather than passing its bar.
2. **View:** one control for the geography (strategic wells or density) plus Mail, Halo, Heat and Route. Each is labelled by the question it answers. A withheld route reads "withheld" in its control, the Golden Path read's own word, and the reason goes in the control's detail. D#19317 §4 said "unavailable", but the canvas still draws the route the graph read carries, so that word would contradict the screen.
3. **Selected node:**
   - label, then kind;
   - the scene's own facts: state (as last ingested), attribution (authored by, assigned to, memory of) and last activity. The landed envelope does not carry `activitySources`, so the section names no source for that time;
   - route rank only while the current route holds the node;
   - the qualified id behind a Copy action;
   - relations grouped by type and direction, with counts.
4. **Open:** routes through one declared vocabulary, `AgentOS.util.GraphNodeSource#sourceOf(node)`. It is a new util beside `GraphSceneEnvelope`, because the layout util is the scene's geometry and overlays (950 of 1,000 lines) and a node's source is identity, not placement.
   - Canonical issue, PR and discussion ids open GitHub.
   - `session:<uuid>` and `summary_<uuid>` open the Memories drill for that session.
   - Every other node names its kind and says it has no source view.
   - Once the scene carries a source column of its own, `sourceOf` reads that column instead.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AgentOS.util.GraphNodeSource#sourceOf(node)` (new) | the scene's `origin#id` grammar (`fleetGraphSceneSource`) | `{type: 'github', url}` for `origin#issue-N`, `#pr-N` and `#discussion-N`; `{type: 'session', sessionId}` for `#session:<uuid>` and `#summary_<uuid>` | `null`, rendered as "no source view" with the node's kind | JSDoc table on the vocabulary | unit arms for every id shape above, and for the non-canonical `ISSUE:N` / `ISSUE:#N` forms, an undeclared kind and a `.` or `..` repository reading `null` |
| Observatory → Memories drill | `cockpit/Controller.mjs#loadSessionMemories` | routes to the cockpit, opens Memories and drills the session | a session the viewer's read cannot return shows the drill's own unavailable or empty state | JSDoc | NL arm |
| the View and Selected-node sections (NL-readable state) | the container's stores and configs | the selection's facts, its source and its grouped relations as data, so peers read what the canvas shows | none | JSDoc | NL arm |

## Acceptance Criteria

- [ ] AC-1 The side panel reads Team, View, Selected node from top to bottom. The strip is gone, and selecting or clearing a node leaves the canvas size unchanged.
- [ ] AC-2 The geography control switches between strategic wells and density and keeps the selection. Each View control's label names the question it answers. A withheld route reads withheld in its control without a hover, with the reason in the control's detail. The control still draws or drops the route the graph read carries, so the word matches what the canvas shows.
- [ ] AC-3 The selected node reads label and kind first; state, attribution and last activity where the scene carries them; and route rank only while the current route holds it. The id sits behind a Copy action and never in the headline. Nothing the scene does not carry renders as known.
- [ ] AC-4 Relations are grouped by type and direction, with counts. Choosing one selects the node at the other end, and the camera does not move (`readStats().camera`).
- [ ] AC-5 Open follows `sourceOf`: canonical GitHub ids open GitHub, a session or summary with a uuid id opens that session's Memories drill, and any other node names its kind and says it has no source view.
- [ ] AC-6 Unit arms for `sourceOf` and the section facts are red on `dev`. An NL arm selects an issue, a session and a concept from the node list by keyboard, and reads label, relations and source action without an id in the headline (§4's falsifier). Goldens in both skins.
- [ ] AC-7 (post-merge, installed) The operator walks Q5 on the installed FM from a cold saved-plane launch: selects an issue, a merged PR and a session and opens each, then selects a concept and reads that it has no source view.

## Out of Scope

- **Evidence for kinds without a source view:** messages, `memory:` nodes, concepts, gaps, strategies, artifacts and classes. Their context needs a viewer-scoped Brain read, because `get_node` descriptions are empty or placeholders for most of them. That leaf joins #312 once its read is defined; until then, this leaf serves the epic's "any node" only for the kinds above.
- **FILE and DIRECTORY links:** most of their paths are not in the engine's tree.
- **The non-canonical issue identities:** 6,032 nodes such as `ISSUE:N` and `ISSUE:#N`, 2,097 of them with a canonical twin. This is a Brain data defect and is noted separately. Here they read "no source view" instead of a guessed link.
- "Working now" (OQ-W8) and editing a peer's colour (agent setup).

## Avoided Traps

- **An inline summary from `get_node`:** measured empty or placeholder for the kinds that matter, so it would print "None." as a summary.
- **A link derived from a non-canonical id or a FILE path:** a link the graph cannot vouch for renders exactly like a real one.
- **Adding the sections inside `ObservatoryContainer`:** it sits 51 lines under the bar.

## Related

#312 (parent) · D#19317 §4 · #320 (Team) · #258 (stable selection)

Decision Record: NOT_NEEDED (D#19317)
Decision Record impact: none

## Signal Ledger
Inherited from #312, graduated from D#19317 at body `updatedAt 2026-09-28T10:56:25Z`: `claude` AUTHOR_SIGNAL (@neo-opus-vega), `gpt` APPROVED (@neo-gpt).

## Discussion Criteria Mapping
- Q5 (what is this, and where's the evidence) → AC-3, AC-4, AC-5, AC-7
- §4 panel hierarchy (Team, View, Selected node) → AC-1, AC-2
- §4 falsifier (a keyboard operator reads label, relations and source action without an id dump) → AC-6

Live latest-open sweep: the latest 20 open Institution issues at 20:34:59Z, plus an org search for "Observatory selected node"; no equivalent, and #312 is the parent. A2A: the last 30 messages, all read states, hold no claim on the Observatory panel. MC rationale sweep: Euclid's Q5 proposal is the only prior art, and no prior decision rejects the shape. Own assignments: #247 only, a different surface.
Structure map: N/A. No `ai/` touch; this is the Institution app layer only.
Origin Session ID: db0e34f7-9d0f-4799-a2c2-3a5033f8bc9a
Retrieval Hint: "Observatory selected node source action sourceOf Memories drill D19317 Q5 panel View section"

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿



## Timeline

- 2026-09-29T20:35:33Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-29T20:35:34Z @neo-opus-vega added the `enhancement` label
- 2026-09-29T20:35:34Z @neo-opus-vega added the `agent-os` label
- 2026-09-29T20:35:34Z @neo-opus-vega added the `ai` label
- 2026-09-29T20:35:34Z @neo-opus-vega added the `design` label
- 2026-09-29T20:35:58Z @neo-opus-vega added parent issue #312
- 2026-09-29T20:37:56Z @neo-opus-vega cross-referenced by #312
- 2026-09-29T21:32:10Z @neo-opus-vega cross-referenced by PR #334
- 2026-09-29T22:21:14Z @neo-opus-vega referenced in commit `5ba9eac` - "fix(agentos): the selected node's id stays behind Copy, and its source never outruns its identity (#333)

- GraphNodeSource#sourceOf reads a kind only from a vocabulary's own entries (`constructor` threw) and
  rejects a `.` or `..` repository, which the browser resolved to a different page.
- The id field leaves the panel's flow and paint but keeps its box, which the copy needs; the focus the
  selection takes returns to the Copy action.
- Six Observatory goldens re-captured; the selected-source arm copies by pointer and by keyboard."
- 2026-09-29T22:42:27Z @tobiu referenced in commit `1d592e1` - "Merge pull request #334 from neomjs/vega/333-observatory-panel

feat(agentos): the Observatory's panel reads the view, then the selected node, which opens its source (#333)"
- 2026-09-29T22:42:28Z @tobiu closed this issue
### @neo-gpt - 2026-09-29T23:44:17Z

## Session handover — separate Engine hide/removeDom investigation

scope: convergent
Origin Session ID: 01a0ee37-7eaa-7d52-9869-ba5d0de51b43

This is follow-up continuity for the Engine observation surfaced during this leaf. It does not reopen `#333` or add a required action to merged `#334`. No Engine ticket or implementation claim exists yet.

**Measured at Engine `dev@745321dec3ca9168f1416c267f4546142ac2ae9c`:**
- `component.Base#hide` silently stamps the child's `vdom.removeDom`, sets the parent's depth to 2, and updates the parent. The method is byte-identical to the Institution's 13.1.0 dependency.
- `mixin.VdomLifecycle` declares `denseUpdate` for silent descendant writes; collection otherwise supplies merged-child IDs to finite-depth tree generation.
- An isolated execution of the actual `util.vdom.TreeBuilder`, with mounted-component records supplied through a process-local manager lookup, gave these controls for four hidden siblings:

| Payload mode | Hidden children |
|---|---|
| depth 2, merged set contains only the head | All four become ignored placeholders; no removeDom markers ship |
| depth 2, no merged set | All four removeDom markers ship |
| depth -1, same merged set | All four removeDom markers ship |

**Limit:** this isolates payload behavior. It does not reproduce the real scheduler/DOM incident, and the missing dense-update boundary remains a hypothesis.

**Next falsifier:** use the real App/VDom harness at current head, hide four direct children while a sibling update merges into the same parent, and record config/vdom/queued IDs/payload/DOM. Carry dense-depth-2 and full-depth positive controls. Only after attribution, run the normal duplicate/ticket/claim gates before a tracked repair.

Do not reuse `#16498` as this bug: its amended contract is window-restoration bystander child loss and the F7 whole-film gate.



