---
id: 243
title: 'The Observatory becomes a left-rail view, not a strip tab'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-ada
createdAt: '2026-09-26T09:34:26Z'
updatedAt: '2026-09-26T12:08:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/243'
author: neo-opus-ada
commentsCount: 0
parentIssue: 9
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-26T12:08:18Z'
---
# The Observatory becomes a left-rail view, not a strip tab

## Context

The operator, 2026-09-26, after #230 put the Observatory into the cockpit: *"the graph is a full-screen view. putting it into the bottom tabs feels wrong => better a left-side main item."* The follow-ups the operator listed for the graph all need room beside the scene: gravity wells, an agent list that filters and highlights one peer's items, highlighting the Golden Path, and a node selection model with details on zoom. A strip tab under the roster does not have that room.

## The Problem

#230 declared the Observatory as a south-strip tab beside "Route graph" (its ledger row: "a south-strip tab"). At the default split the strip is about 280 px tall at 1280×800, so the 3D scene opens as a letterbox under the roster, and every follow-up would have to fit into that band.

## The Architectural Reality

- `apps/agentos/view/Viewport.mjs`: the left rail's keeper views (Home, Fleet, System, Accounts, Chat) are the shell's tab items, each with a `header` `{iconCls, route, text}`. `ViewportController` routes `/home`, `/fleet`, `/system`, `/accounts`, `/chat`.
- `apps/agentos/view/fleet/cockpit/Container.mjs` holds the `observatory` pane in the dock catalog, and `apps/agentos/util/CockpitPerspectives.mjs` places it in `stream-tabs`.
- The renderer is app-space today (`apps/agentos/canvas/Observatory.mjs` on the canvas worker). The scene reads the cockpit's Golden Path envelope.
- A keeper view sits outside the cockpit's provider, so the `goldenPathEnvelope` leaf moves up to the Viewport provider. The cockpit's read still writes it, through setData's closest-owner walk. Consumers that read it by name keep working. The Route graph's `GraphCanvas`, however, serializes the bound envelope for the canvas worker. From the cockpit's child scope that serialization came out empty — an engine gap in how the hierarchical proxy enumerates inherited data, `neomjs/neo#19266`, bisected. This ticket therefore depends on an Institution engine pin that includes that fix.

## The Fix

1. A left-rail keeper view "Observatory" (route `/observatory`) hosts the scene full-size, with room for the follow-ups' side list.
2. The strip loses its Observatory tab. The textual Golden Path stays in the strip, where it is read beside the roster.
3. The scene reads the same Golden Path state from the new view: one read owner, no second fetch.
4. The Institution engine pin moves to a `dev` commit that includes `neomjs/neo#19266`.

## Acceptance Criteria

- [ ] AC-1 The rail shows an Observatory item; `/observatory` renders the scene at the view's full size, and the strip has no Observatory tab (NL arm).
- [ ] AC-2 The scene resizes with the window and keeps orbit/zoom working there (the #230 AC-2 arm moves to the view).
- [ ] AC-3 Switching views away and back keeps the scene without a second Golden Path read (unit or NL arm on the read owner).
- [ ] AC-4 After the leaf moves, the strip's Route graph still draws the current route (its visual arms), on the bumped engine pin.

## Out of Scope

- The follow-ups themselves (filters, highlight, selection, gravity wells): their own tickets.
- The 2D "Route graph" tab's future: the Observatory's author decides whether it retires once the view covers it.
- The engine renderer extraction (`neomjs/neo#10034`'s engine half).

## Related

#9 (parent: keeper views · nav) · #230 / #234 (the Observatory) · #10 · `neomjs/neo#19151` (H3) · `neomjs/neo#19266` (the engine dependency)

Owner: @neo-opus-ada (lane-claimed 2026-09-26).

Live latest-open sweep: the latest 20 open issues at 2026-09-26T09:33:49Z — no equivalent. A2A in-flight sweep (all read states, last 60 min): no claim on the Observatory's placement. Memory Core sweep: #230's own placement row is the only prior decision. Own-assignment sweep: none open (#235 closed with #236 at 09:31Z).

Origin Session ID: 1b945fcf-1142-475f-8007-ac18d51c069a
Retrieval Hint: `query_raw_memories("Observatory left rail keeper view full-screen graph strip tab")`


## Timeline

- 2026-09-26T09:34:27Z @neo-opus-ada added the `enhancement` label
- 2026-09-26T09:34:28Z @neo-opus-ada added the `agent-os` label
- 2026-09-26T09:34:28Z @neo-opus-ada added the `ai` label
- 2026-09-26T09:34:28Z @neo-opus-ada added the `design` label
- 2026-09-26T09:35:01Z @neo-opus-ada added parent issue #9
- 2026-09-26T10:45:53Z @neo-opus-ada cross-referenced by #252
- 2026-09-26T10:47:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-26T11:23:11Z @neo-opus-ada cross-referenced by #19266
- 2026-09-26T11:28:58Z @neo-opus-ada cross-referenced by PR #19269
- 2026-09-26T11:43:50Z @neo-opus-ada referenced in commit `6299fce` - "build(deps): engine pin 12 → dev@a87a89e2 carries inherited provider-data enumeration (#243)

The Observatory keeper view moves the goldenPathEnvelope leaf up to the
Viewport provider. The cockpit's Route graph serializes that leaf for the
canvas worker, and from the cockpit's child scope the serialization was empty
until neomjs/neo#19266 made the hierarchical proxy enumerate the parent chain.
27 engine commits ride along (87ac80a6..a87a89e2)."
- 2026-09-26T11:43:50Z @neo-opus-ada referenced in commit `bcf3a65` - "test(agentos): the nav family pins re-capture the rail and the strip, and the baseline stamp is re-issued (#243)"
- 2026-09-26T11:45:38Z @neo-opus-ada referenced in commit `c04fa6a` - "test(agentos): the Observatory keeps its scene across a view round trip and resizes with the window (#243)"
- 2026-09-26T11:46:13Z @neo-opus-ada cross-referenced by PR #253
- 2026-09-26T12:08:18Z @tobiu referenced in commit `0f32ec4` - "Merge pull request #253 from neomjs/ada/243-observatory-view

feat(agentos): the Observatory is a keeper view in the shell rail (#243)"
- 2026-09-26T12:08:18Z @tobiu closed this issue

