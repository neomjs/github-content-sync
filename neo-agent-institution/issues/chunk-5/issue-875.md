---
id: 875
title: Home's field misses a reader switch that renders after 30 ms
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-11-04T23:28:58Z'
updatedAt: '2026-11-05T00:11:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/875'
author: neo-opus-ada
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-05T00:11:33Z'
---
# Home's field misses a reader switch that renders after 30 ms

## Context

CI run 41904834413 (PR #874, 2026-11-04) failed `test/playwright/e2e/agentos/HomeField.spec.mjs` › "a reader switch and a resize both move the lines, and the field follows them (AC-2)" at "the first-run reader". The field stayed quiet under the returning reader's lines, a one-line `{width 174.3, height 27.2}` at y 261.4 where the first-run lede stood at `{width 660, height 54.4}`. The rerun passed, and the arm passes 8/8 locally at that head.

Reproduced on purpose: a test driver that delays every vdom batch the App worker sends (`Neo.vdom.Helper.updateBatch`) by 150 ms across the switch fails the arm 2 of 2 times with the same signature. At 0 ms it passes 2 of 2. Throttling the page's CPU 6× does not reproduce it, because the race sits on the worker round trip, not on the main thread.

## The Problem

`Container#applyState` changes the hero's lines, then calls `HomeCanvas#measureQuiet()`, which reads the lines' rects 30 ms later. A change reaches the DOM only when its render lands: a text through its own component, and a shown or hidden line through the lines' container. When that takes longer than 30 ms, the read gets the old boxes and hands them to the renderer. Nothing measures again until the canvas resizes, so the field stays quiet under lines that are gone and draws over the ones that replaced them. On CI that takes a slow runner; in the product, a busy machine.

## The Architectural Reality

- `apps/agentos/view/home/Container.mjs` `applyState()`: the operator line's vdom plus `update()`, `lead.set()`, the `hidden` flags of lede, first-run and doors, and `plane-line.set()`, then `canvas?.measureQuiet()`.
- `apps/agentos/view/home/Canvas.mjs` `measureQuiet()`: `await me.timeout(30)`, then `getDomRect(quietIds)`.
- Engine: `component.set()` returns the component's `promiseUpdate()`. `hide()` and `show()` update the parent and return nothing. `promiseUpdate()` "Renders this component's current vdom and settles once that render has landed in the DOM" (`src/mixin/VdomLifecycle.mjs`).
- Design authority: `measureQuiet`'s docblock, "Measures the quiet components one frame after the call, so a line the Home view just changed has its new box", and #365's Fix, "after `applyState` changes the lines". The 30 ms timer is the guess that breaks that intent.

## The Fix

`applyState` keeps the renders that carry its changes: the operator line's `promiseUpdate()`, the lead's and the plane line's `set()` promises, and the lines' container's `promiseUpdate()` for the shown and hidden lines. It measures once all of them have landed. `measureQuiet`'s docblock then names what its 30 ms is for: coalescing a burst of calls.

## Acceptance Criteria

- [ ] e2e: a HomeField arm switches the reader while the App worker's vdom batches are delayed 150 ms by a test driver, and the field follows the first-run reader's lines. Red on `dev` first.
- [ ] The rest of `HomeField.spec.mjs` stays green.
- [ ] The visual goldens hold and the baseline stamp is current.

## Out of Scope

- A web font that loads after a measurement, with no resize and no `applyState` to follow it. It gets its own ticket if seen.
- An engine change making `hide()` or `show()` return their render.

## Related

#365 (the quiet field), #244 (Home's canvas), PR #874 (where it surfaced).

Live latest-open + closed sweep: latest 20 open Institution issues at 2026-11-04T23:28:27Z, closed search "HomeField OR measureQuiet OR quiet rects OR reader switch" (#365, #244, #366; #365's Out of Scope and Avoided Traps read, no exclusion); no equivalent. A2A sweep: last 30 and the newest 12 at 23:29Z, no claim on Home's field. MC sweep: "HomeField e2e flaky quiet rects measureQuiet stale after reader switch", a 2026-10-10 mention of a HomeField CI flake and no ticket. Own-assignment: #841, #840, #516, #424, none on Home.

Origin Session ID: 2171b2cd-e447-4cac-8b9f-ccadc6e20db5
Retrieval Hint: "HomeField flake measureQuiet 30 ms reader switch renders late quiet rects stale"

## Timeline

- 2026-11-04T23:28:59Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-04T23:29:00Z @neo-opus-ada added the `bug` label
- 2026-11-04T23:29:01Z @neo-opus-ada added the `ai` label
- 2026-11-04T23:29:01Z @neo-opus-ada added the `testing` label
- 2026-11-04T23:43:39Z @neo-opus-ada cross-referenced by PR #876
- 2026-11-05T00:11:32Z @tobiu referenced in commit `6bcd214` - "fix(agentos): Home's field measures a reader switch once its render lands (#875) (#876)

applyState measured the hero's lines on a 30 ms timer, so a render that took longer landed after the read and the field stayed quiet under the old lines until a resize. It now renders the lines' container over its whole subtree and measures once that render has landed. No hidden line's own render is awaited, since that waits for a mount. A late-render test driver holds the App worker's vdom batches, and a new HomeField arm switches the reader under a 150 ms delay. The visual stamp moves for the two Home files; no golden changed."
- 2026-11-05T00:11:34Z @tobiu closed this issue

