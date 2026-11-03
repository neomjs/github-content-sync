---
id: 869
title: Awaiting-merge menu overflows when many PRs are approved
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - agent-os
  - ai
assignees: []
createdAt: '2026-11-03T17:24:26Z'
updatedAt: '2026-11-03T17:24:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/869'
author: neo-gpt-euclid
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
# Awaiting-merge menu overflows when many PRs are approved

## Context

Operator-reported Fleet Manager defect, 2026-11-03, with a screenshot captured at 15:16 CET: the Fleet header reads **33 awaiting merge**, but opening the list produces a tall strip along the right edge with PR text clipped horizontally and content extending beyond the visible area. The operator cannot use it as a readable merge queue.

The screenshot is a cropped observation, not a full-window geometry receipt. Its build pins, complete viewport size and exact failure threshold are unknown. **33 is the observed queue size, not a magic cutoff.** Rows are visibly present; missing records or a data-fetch failure have not been established.

## The Problem

The queue must remain usable when the team has more approved PRs than fit vertically. Reproduce with 33 representative records, including long titles, in the existing rendered Fleet fixture; open the header chip at normal and reduced window sizes. Capture menu/viewport rectangles, computed sizing and scroll dimensions before choosing the fix.

Evidence scope: operator screenshot plus source inspection; no independent installed runtime reproduction yet.

## The Architectural Reality

At [Institution `5046f84f`](https://github.com/neomjs/neo-agent-institution/commit/5046f84f7a215284d1c1fee24bdbace092f7c4cc), the three implementation blobs below match the inspected checkout:

- [`AwaitingMergeButton.ensureMergeMenu`](https://github.com/neomjs/neo-agent-institution/blob/5046f84f7a215284d1c1fee24bdbace092f7c4cc/apps/agentos/view/fleet/roster/AwaitingMergeButton.mjs#L153) creates the floating menu over the provider-owned Store, without size/alignment overrides.
- [`AwaitingMergeMenuList`](https://github.com/neomjs/neo-agent-institution/blob/5046f84f7a215284d1c1fee24bdbace092f7c4cc/apps/agentos/view/fleet/roster/AwaitingMergeMenuList.mjs#L13) extends `Neo.menu.List`; its own config supplies no height bound.
- [`AwaitingMergeButton.scss`](https://github.com/neomjs/neo-agent-institution/blob/5046f84f7a215284d1c1fee24bdbace092f7c4cc/resources/scss/src/apps/agentos/fleet/roster/AwaitingMergeButton.scss#L42) supplies 240px minimum / 360px maximum width, with no height cap; titles clamp to two lines.
- Existing [rendered queue coverage](https://github.com/neomjs/neo-agent-institution/blob/fd5191c3d2a4cd916f19d5621170a10d35816ddf/test/playwright/visual/FleetCockpitVisual.spec.mjs#L234) uses two PRs and checks styling/title clamping, not a queue exceeding available height.

A natural-height menu exhausting the engine's available alignment space is a plausible explanation for the right-edge placement. **Confirm it from runtime measurements; it is not the established cause.**

Design authority: #505 requires “No collapsed overlays, no clipped rows” and content readable in full. The menu's JSDoc names it the operator's PR list and retains provider Store ownership.

## The Fix

Keep this as one menu-rendering repair. Bound the floating list to the available viewport and allow overflow to scroll within it, preserving a readable width and placement. Use the existing `Neo.button.Base` / `Neo.menu.List` sizing, scrolling and alignment primitives; keep skin in the owning SCSS. Change a shared engine primitive only if a reduced runtime witness establishes a shared defect.

Add a rendered regression to `FleetCockpitVisual.spec.mjs`, alongside the existing queue cases. Use `npm run test-visual -- test/playwright/visual/FleetCockpitVisual.spec.mjs`; keep relevant behavioral checks in the existing `awaitingMergeButton.spec.mjs` suite via `npm run test-unit`.

## Contract Ledger

| Surface | Authority | Required behavior / fallback | Docs | Evidence |
| --- | --- | --- | --- | --- |
| Floating queue | #505; existing menu composition | A queue taller than the available space stays inside the viewport and scrolls internally | Existing menu JSDoc if changed | Menu/viewport bounds and scroll measurements |
| PR rows | #512; `createItemContent` | Titles, reference fallback, stale chip and forge link stay intact | Existing row JSDoc | First/last row reachability and link checks |
| Small queue and dismissal | Existing button/menu behavior | Normal placement and existing keyboard/Escape/outside dismissal remain usable | Existing component contract | Existing tests plus small-queue control |

Decision Record impact: none; this repairs the existing view contract.

## Acceptance Criteria

- [ ] A rendered queue of 33 representative PRs, tall enough to exceed available height, opens fully inside the available FM viewport with readable row width, without the clipped right-edge strip or outer horizontal overflow.
- [ ] Internal scrolling reaches the first and last PR; both retain usable forge links. Records are not silently dropped or capped to hide overflow.
- [ ] Reopening after reducing the window size preserves those outcomes. Record actual viewport/menu dimensions and test both skins.
- [ ] A small-queue control retains title wrapping, reference fallback, stale chips and existing dismissal behavior.
- [ ] The regression asserts menu bounds and last-row reachability, and fails on the pre-fix implementation.
- [ ] **Post-merge installed check:** on the next candidate carrying the repair, record Institution/Brain/Engine pins, viewport dimensions and a screenshot showing the large queue readable and its last row reachable. Source tests do not retire this installed check.

## Out of Scope

PR eligibility, counts or producer schemas; changing merge authority; automatic merging; redesigning the queue; unrelated switcher defects or Stage launch behavior.

## Avoided Traps

Do not fix this by hiding records, requiring a larger window, loosening screenshot tolerances, or treating a static two-row golden as large-queue evidence.

## Related and Ownership

Parent: #505. Related: #512 (titles), #743 (token/link styling), #500 (distinct collapsed instance-switcher incident).

unowned-rationale: Ready for a contributor; maintainers provide planning and review. This operator-requested bug filing does not claim implementation.

## Creation Checks

Live latest-open + closed sweep: latest 20 open issues, open PRs, and closed searches for `awaiting merge` / `menu overflow` checked on 2026-11-03 at 17:22–17:25 UTC; same-surface #512/#743/#500 bodies and exclusions read. No equivalent large-queue repair found.
A2A: latest 30 messages, all read states; no overlapping claim.
MC sweep: three symptom framings plus `AwaitingMergeMenuList too many approved PRs list fails to render`; no prior mapping of this failure found.
Own-assignment sweep: two open Institution issues (#657, #477), neither owns menu sizing.
Structure map: N/A — existing Institution view/SCSS/test surface; no Brain, MCP, config or placement changes proposed.

Retrieval Hint: "FM awaiting merge 33 approved PRs tall list right edge clipping AwaitingMergeMenuList"


## Timeline

- 2026-11-03T17:24:27Z @neo-gpt-euclid added the `bug` label
- 2026-11-03T17:24:28Z @neo-gpt-euclid added the `good first issue` label
- 2026-11-03T17:24:28Z @neo-gpt-euclid added the `help wanted` label
- 2026-11-03T17:24:28Z @neo-gpt-euclid added the `agent-os` label
- 2026-11-03T17:24:28Z @neo-gpt-euclid added the `ai` label
- 2026-11-03T17:24:47Z @neo-gpt-euclid added parent issue #505

