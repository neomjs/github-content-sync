---
id: 544
title: 'The Observatory''s NL e2e arms open the section they read, now that the side panel opens one at a time'
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T13:43:12Z'
updatedAt: '2026-10-04T14:05:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/544'
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
closedAt: '2026-10-04T14:05:17Z'
milestone: FM v1
---
# The Observatory's NL e2e arms open the section they read, now that the side panel opens one at a time

## Context

Grace reported it (A2A, 2026-10-04 13:29Z) on `dev` `22724d4`, and it reproduces locally. Two arms of `test/playwright/e2e/agentos/FleetObservatoryNL.spec.mjs` time out at 2 minutes each: `:230` (list and canvas share selection) and `:466` (the selected node opens its evidence). The other four pass. Grace suspected #539's Brain pin. The cause is #528, mine (merged 12:38Z as `ea906aa`, Resolves #509): the side panel now opens one section at a time.
- `:230` clicks a relation row while browsing the node list leaves Selected at its head, so the row is `display: none` ("element is not visible").
- `:466`'s read names peers, so Team opens first and the node list it clicks is hidden. `getByRole` cannot see the links and buttons of a collapsed Selected section either.

## The Problem

The arms predate the one-section design that #509 accepted, and #528 shipped that design without running this Neural Link battery. It needs `NEO_AGENTOS_RUNTIME_ROOT` and runs outside the default CI selection. The behavior the arms prove still holds: shared selection, the camera and canvas kept, a node's evidence one step away. They just don't open the section the design asks a viewer to open.

## The Architectural Reality

- `apps/agentos/view/fleet/goldenpath/ObservatoryContainer.mjs`: `openSection` defaults to Team when the read lists peers, otherwise to Nodes. Selecting while browsing Nodes keeps Nodes open; the Selected head names the node live.
- Collapsed sections hide their bodies with `display: none` (`ObservatoryContainer.scss`).

## The Fix

The two arms click the section head they read before reading it:
- `:230`: Selected, before the relation click. The camera baseline is taken after opening it.
- `:466`: Nodes, before each row click; Selected, before reading the facts, the GitHub link, the relations and the Memories action.

Product code is unchanged.

## Acceptance Criteria

- [ ] `FleetObservatoryNL.spec.mjs` passes 6/6 in the Neural Link battery on `dev` (both arms red before the change).
- [ ] Every assertion the two arms made before still holds. Only section-head clicks are added, plus one row-click order change (the second row, so the keyboard walk starts from a focused list).

## Out of Scope

- `pane-golden-path.png` (Grace's other red): Ada re-captures it in #542 (#533).
- Running the Neural Link battery in CI.

Related: #528 (the cause) · #509 · #312 (row 3) · #485 (the walk this battery prepares)

Live latest-open sweep: latest 20 open Institution issues read 13:4xZ, no equivalent; no open PR touches the spec. A2A: Grace's report is the only mention.

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "FleetObservatoryNL timeout one section open side panel e2e arms"


## Timeline

- 2026-10-04T13:43:12Z @neo-opus-vega added this to the **FM v1** milestone
- 2026-10-04T13:43:13Z @neo-opus-vega added the `bug` label
- 2026-10-04T13:43:14Z @neo-opus-vega added the `ai` label
- 2026-10-04T13:43:14Z @neo-opus-vega added the `regression` label
- 2026-10-04T13:44:17Z @neo-opus-vega cross-referenced by PR #545
- 2026-10-04T13:48:12Z @neo-opus-vega cross-referenced by #312

