---
id: 454
title: The memories pane's edge replay retires once the engine pin re-arms a mounted body
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-02T16:40:29Z'
updatedAt: '2026-10-02T18:14:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/454'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 455 Carry the Engine mount-edge fix with Fleet pop-out compatibility'
blocking: []
closedAt: '2026-10-02T18:14:08Z'
---
# The memories pane's edge replay retires once the engine pin re-arms a mounted body

## Context

PR #434 (#429) moved the memories pane's continuation to the engine's `scrollEdge`. Sophie's review found that a summary window landing while a drill owns the zone runs the hidden register's layout, announces the new count and latches it in `Neo.grid.Body`, so the list shown again at that count never announced; the pane answered with `summaryEdgeDeferred` (`apps/agentos/view/fleet/memories/Container.mjs`): remember the refused edge, replay it once in `onDrillBackClick`.

neomjs/neo#19361 / PR neomjs/neo#19364 (merged 2026-10-02, `e35f4bddb0`) put the fix where it belongs: a body that mounts clears its latch, so its next layout at the edge announces again. With an Institution engine pin carrying that commit, the pane's replay is a second copy of an engine behaviour.

## The Problem

Two mechanisms answer one re-show: the pane's replay asks on back, the engine's re-announcement asks on the register's first layout after it mounts; the pane's in-flight gate makes them one request today, but the bookkeeping (`summaryEdgeDeferred`, three writes, one replay) stays in the pane with no reason left.

## The Fix

With the pin on `dev`: delete `summaryEdgeDeferred` and its writes (`onSummaryScrollEdge`'s deferral branch, `onDrillBackClick`'s replay, `afterSetActiveAgent`'s and `applySnapshot`'s resets); `onSummaryScrollEdge` refuses behind a drill, nothing more. The two drill arms in `memories/container.spec.mjs` keep their assertions and drive the engine's own mount: `summaryGrid.body.mounted = true` before the re-show layout stands for the register's re-mount in the browser, and the request at the new depth must come from the engine's re-announcement alone.

## Acceptance Criteria

- [ ] `summaryEdgeDeferred` and every write to it are gone from `memories/Container.mjs`; the pane's docblock and `onSummaryScrollEdge`'s say "refuses behind a drill" only.
- [ ] The two drill arms stay green with the request at the new depth produced by the engine's re-announcement after `mounted = true`, and red when that mount write is removed from the arm (the engine, not the pane, is the source).
- [ ] The memories NL journey and the rest of the memories arms stay green; `pane-memories.png` unchanged.

## Out of Scope

- The engine pin itself (the pin lane cuts it; this leaf cannot start before it is on `dev`).
- The mailbox pane (never hides its grid; carries no replay).

## Related

#429 / PR #434 (the replay) · neomjs/neo#19361 / PR neomjs/neo#19364 (the engine's re-arm) · #433 (the precedent: a consumer's duplicate of an engine seam retired with the pin)

Owner: Vega (self-assigned on creation); blocked by the next engine pin ticket once it exists — the native dependency follows its creation.

Live latest-open sweep: the open Institution queue at 16:41Z has no leaf on the memories pane's edge or replay; `gh issue list --search "memories edge OR summaryEdgeDeferred OR replay"` returns nothing besides pin tickets.

Origin Session ID: 60d9be31-4233-40fd-9b07-6ee6a9ebf6cf
Retrieval Hint: "memories pane summaryEdgeDeferred replay retire engine pin re-arm mounted scrollEdge"


## Timeline

- 2026-10-02T16:40:31Z @neo-opus-vega added the `enhancement` label
- 2026-10-02T16:40:31Z @neo-opus-vega added the `agent-os` label
- 2026-10-02T16:40:32Z @neo-opus-vega added the `ai` label
- 2026-10-02T16:40:50Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-02T17:15:48Z @neo-gpt-emmy cross-referenced by #455
- 2026-10-02T17:16:40Z @neo-gpt-emmy marked this issue as being blocked by #455
- 2026-10-02T17:30:27Z @neo-gpt-emmy cross-referenced by PR #458
- 2026-10-02T17:54:58Z @neo-opus-vega cross-referenced by PR #459
- 2026-10-02T18:14:09Z @tobiu referenced in commit `904b262` - "refactor(agentos): the memories pane drops its edge replay, the engine re-arms a mounted register (#454) (#459)

neo #19364, carried by the #458 pin, clears grid.Body's scrollEdge latch when a body mounts, so the summary register shown again after a drill announces the edge a continuation landed behind it. summaryEdgeDeferred, its writes and the back replay go; onSummaryScrollEdge refuses behind a drill and nothing more. The drill arms set the register's mounted flag before the re-show layout and go red without it."
- 2026-10-02T18:14:09Z @tobiu closed this issue

