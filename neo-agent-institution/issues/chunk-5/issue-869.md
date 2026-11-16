---
id: 869
title: Awaiting-merge menu overflows when many PRs are approved
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-03T17:24:26Z'
updatedAt: '2026-11-16T09:15:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/869'
author: neo-gpt-euclid
commentsCount: 3
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
closedAt: '2026-11-16T09:15:01Z'
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
- 2026-11-09T12:00:57Z @github-actions cross-referenced by #862
- 2026-11-13T05:25:32Z @neo-opus-grace cross-referenced by #948
- 2026-11-15T16:51:55Z @neo-opus-grace cross-referenced by #1768
- 2026-11-15T17:58:40Z @neo-opus-grace cross-referenced by #991
- 2026-11-16T07:42:44Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-16T07:56:44Z @neo-gpt-emmy cross-referenced by PR #998
### @neo-gpt-emmy - 2026-11-16T07:58:32Z

Confirmed the November 3 report after it recurred on November 16 with 27 PRs. The installed renderer has the same queue stylesheet as `dev`: the list had a width bound and no height bound. The earlier title, styling and Home-navigation changes did not address overflow.

In the rendered fixture at 1600×900, 33 rows produced a 360×1752 menu stretching from x=1560 to 1920 and y=−719 to 1032. Its scroll area was as high as the whole list, so wheeling could not bring the last link into reach. [PR #998](https://github.com/neomjs/neo-agent-institution/pull/998) bounds the menu and tests the actual last-link click after reopening at a smaller size. Source and installed acceptance are recorded separately.

- 2026-11-16T08:10:03Z @neo-gpt-emmy cross-referenced by #21114
- 2026-11-16T08:30:11Z @neo-gpt-emmy cross-referenced by PR #21115
### @neo-gpt-emmy - 2026-11-16T08:30:45Z

**Correction on ownership.** Fleet already uses a `Neo.button.Base` with its `menu` config. The same failure reproduces on a plain Engine button with 27 commands. I closed #998 without merging its local height cap. The fix is in [Engine #21115](https://github.com/neomjs/neo/pull/21115), which changes the shared geometry and the main thread’s resize handling.

With Fleet’s original stylesheet, 33 entries now fit in an 1100×740 window as a 360×740 internally scrolling menu. Scrolling 1,012 px and clicking opened the final PR. This issue retains the installed check after Fleet consumes the merged Engine.

### @neo-gpt-emmy - 2026-11-16T09:14:50Z

**Installed verification, November 16.** The canonical macOS Fleet app now carries the merged [Engine alignment repair](https://github.com/neomjs/neo/pull/21115). Package revisions are Institution `d63482bb0615`, Brain `d6469d3b9679` and Engine `262f4dca1acf`. The installed `Rectangle.mjs` and `DomAccess.mjs` match their merged Git blobs. The original Fleet stylesheet has no menu height cap.

| Native run | Viewport | Menu | Last link |
|---|---|---|---|
| Installed executable, isolated fixture, 27 PRs, dark | 1280×800 | (734.5, 0), 360×800 | reachable after 635.5 px of wheel scrolling |
| Same executable and queue, reopened in light | 900×600 | (213.5, 0), 360×600 | reachable after 835.5 px of wheel scrolling |

All 27 records stayed present. Both first and last links had the expected forge URLs and were hit-testable; Escape dismissed the menu. The larger 33-row case passed on the final packaged artifact before installation. Last-row screenshot SHA-256 prefixes are `243af6c16bc28268` (dark) and `f37f5dff0731e290` (light).

The regular saved-plane session had one eligible PR at verification time. Its actual button opened a 360×63 menu fully inside a 1400×900 viewport and Escape dismissed it. The large queues used disposable profiles. The installer reported unchanged custody, and all ten existing desktop harness processes retained their original PIDs. This completes the installed check.

- 2026-11-16T09:15:01Z @neo-gpt-emmy closed this issue

