---
id: 103
title: 'The Review preset''s detail pane stays hidden after the FLIP settle — red 3/3 at engine pin 6, headed too; #66 is closed and the hold has no owner'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T14:57:51Z'
updatedAt: '2026-09-13T12:23:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/103'
author: neo-fable-clio
commentsCount: 5
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
closedAt: '2026-09-13T12:23:59Z'
---
# The Review preset's detail pane stays hidden after the FLIP settle — red 3/3 at engine pin 6, headed too; #66 is closed and the hold has no owner

## Context

`FleetCockpitDockNL.spec.mjs:199` ("perspective presets switch the committed document") holds one step the cockpit cannot pass on engine `dev@205bc52f8a` (#98, pin 6): after `activatePerspective('Review')` commits the `[0.45, 0.55]` document and `docReview.items.detail.autoHidden` reads `false`, the materialized detail pane `.fm-agent-detail` stays hidden — `expect(page.locator('.fm-agent-detail')).toBeVisible({timeout: 10000})` reports the node present with an unexpected `hidden` visibility. Measured 2026-09-04 while pinning: red **3/3** headless at pin 6 and red **headed** as well; **1/3** at `ab5235e69a` (one engine commit earlier, with a freshly built theme CSS); 2/2 green headless on 2026-09-02 (README) and green in the 2026-09-04 morning battery on pin 5.

The spec carries the hold as a comment ("Known Engine hold, kept failing-honest rather than skipped: … the projection stages tab chrome with `hideMode: 'visibility'` and un-hides it on the FLIP settle. Under HEADLESS Chromium the settle never lands … Ledger: neo-agent-institution#66"). #66 was closed COMPLETED on 2026-09-01 — the hold has no open owner, and the README's battery paragraph names it as one of two stated reds. This ticket is that owner.

Live latest-open sweep: checked the latest 20 open issues at 2026-09-04T14:53Z; no open issue names the FLIP settle, the Review preset or the detail pane (search over open issues: none). A2A claim sweep: no claim on this arm. Memory Core: the hold's prior record is #66 (closed) and the spec comment.

## The Problem

The Review preset's detail pane is a "genuinely absent" item that the dock projection materializes on the perspective switch; its tab chrome is staged with `hideMode: 'visibility'` and un-hidden on the FLIP settle. When the settle does not land within the wait, the pane is mounted but `visibility: hidden` — the operator's Review perspective opens with an empty detail slot. On pin 5 the settle landed often enough to read green; at pin 6 it lands 0/3 headless and 0/1 headed, so the product surface changed for the worse with the engine delta (the dock commits in that delta: neomjs/neo#18253 — a workspace answers the multi-window restore seams; the projection files themselves are unchanged between the pins, per `git diff --stat 35d468b51f..205bc52f8a -- src/dashboard/dock`).

## The Architectural Reality

- The witness lives in the NL battery (`test/playwright/e2e/agentos/FleetCockpitDockNL.spec.mjs`, lines ~250–290); the cockpit's document and presets in `apps/agentos/util/CockpitDockDocument.mjs`; the engine's dock projection (`src/dashboard/dock/projection/*`, `Workspace.mjs`) owns the staging/un-hide on settle.
- Whether the settle is the engine's FLIP pipeline not completing under the pinned Chromium or the cockpit's own projection wait is the first thing to measure — the arm's own comment guessed the engine; nobody has measured it since #66.
- The pin PR (#102) ships with this arm failing-honest and this ticket as its owner; the README battery paragraph states it the same way.

## The Fix

Measure first: instrument the settle (does the FLIP transition end fire? is the un-hide scheduled on it? what does the pane's computed visibility read at 1s / 5s / 10s after the switch, headless and headed, on pin 6 vs `ab5235e69a`?). Then either repair the cockpit's side (a materialization that does not depend on a transition end to un-hide), or file the engine leaf with the measurement and keep the arm red-honest until it lands.

## Acceptance Criteria

- [ ] AC-1 The settle failure is attributed with a measurement (engine FLIP pipeline vs cockpit projection wait), recorded on this ticket.
- [ ] AC-2 `FleetCockpitDockNL:199` green headless under the Brain root at the current pin, 3/3 on three consecutive runs, and green headed once.
- [ ] AC-3 The README battery paragraph no longer states this red; the spec comment names the resolution instead of the hold.

## Out of Scope

- The other NL reds (the A11y reorder step is #99 / PR #101).
- The rail-drawer witness's Escape dismissal, which rides the same settle — re-measure it here only if AC-1 implicates the engine pipeline.

## Related

#98 / PR #102 (the pin that measured it) · #66 (closed; the previous ledger) · neomjs/neo#18253 (the dock commit in the pin delta) · `FleetCockpitDockNL.spec.mjs:276-285` (the hold comment).

Ownership: unowned — the pin lane states it; the measurement is a separate leaf any peer can take.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("Review preset detail pane hidden FLIP settle pin 6 FleetCockpitDockNL headed red")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Deferred here from neomjs/neo#18297 AC-2 (`[L3-deferred]`, 2026-09-04)

The browser half of the engine fix's AC-2 is owed by this ticket, verbatim: with the engine pin that carries PR neomjs/neo#18321, **one shell in the host after the second back-to-back commit, visible at `shellIndex`; every pane of the committed document reachable; N=5 headless** — observed by the existing arm `test/playwright/e2e/FleetCockpitDockNL.spec.mjs:199` under the Brain root. If that arm does not yet assert pane reachability or the shell's visibility at `shellIndex` explicitly, the assertions land with the pin bump; the 5/5 receipt in the PR body closes this ticket and AC-2's browser half together. Lineage: Vega's proposal https://github.com/neomjs/neo/issues/18297#issuecomment-5546977191, applied in https://github.com/neomjs/neo/issues/18297#issuecomment-5547135227.


## Timeline

- 2026-09-04T14:57:52Z @neo-fable-clio added the `bug` label
- 2026-09-04T14:57:53Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T14:57:53Z @neo-fable-clio added the `ai` label
- 2026-09-04T14:57:53Z @neo-fable-clio added the `testing` label
- 2026-09-04T14:59:38Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-04T15:25:57Z @neo-fable-clio referenced in commit `436cd6d` - "docs(agentos): README battery truth — one stated red (#103); the A11y reorder step is green on the pin-6 Engine (#99)"
- 2026-09-04T15:26:36Z @neo-fable-clio cross-referenced by PR #101
- 2026-09-04T15:37:53Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-04T15:51:14Z

## AC-1 — the settle failure, measured and attributed (2026-09-04, engine pin 6 `205bc52f8a`, headless Chromium under the Brain root)

**Rate.** The arm itself (`FleetCockpitDockNL.spec.mjs:199`, `--repeat-each 5`): **3 of 5 red** (each red at the 10s `toBeVisible` timeout; the greens land in 1.8s). An untracked probe spec with the arm's exact timing (Focus click → poll the commit → `activatePerspective('Review')` at once → poll the commit → passive 250ms ticks reading DOM styles and worker configs over the Neural Link for 30s): **4 of 5 never visible within 30s**. The same probe with ~150ms of observation between the Focus commit and the Review switch (a `page.evaluate` that drives rAF for 100ms): **0 of 8** — the window is the gap between the two refreshes, which is why the battery read 34/34 one run and 3/3 red another.

**Where the pane is hidden.** Playwright's failure snapshot and every failing probe tick agree: the workspace holds **two edge-zone shells** — the Focus projection's `#neo-container-94` (visible) and the Review projection's staged `#neo-container-98` with inline `visibility: hidden`; the detail pane (`#neo-fm-agent-detail-1`, 307px wide, `neo-active-item`) sits inside the hidden one. Neither tab chrome nor the pane carries the hide; the staged shell does.

**Worker side, for the whole 30s** (`getInstanceProperties` on each tick): `neo-container-98` → `hidden: true, hideMode: 'visibility', mounted: false`; `neo-container-94` → `hidden: false, mounted: true`; the cockpit host → `updateDepth: -1`. The reconciler never reached its un-hide (`nextShell.setSilent({hidden: false})`): the staged shell's own insert update never completed. This is not a DOM flush that went missing and not a FLIP settle: rAF is live (6–7 frames per 100ms on every tick), a deliberate 100ms rAF nudge at 10s changes nothing, `neo-dashboard-dock-animating` is gone by tick 1, `document.getAnimations()` reads 0 from tick 2, no page errors, and the FLIP addon's cleanup restores geometry/opacity/transform only, never `visibility`.

**The cause, in the App worker's console** (16 occurrences across the 4 failing runs, none in the passing one):

```
vdom update failed neo-fm-fleet-cockpit-1 util.VNode.getVnode: Component not found for id: neo-tab-header-button-13
    at VNode.getVnode (src/util/VNode.mjs:212)  at VNode.createMap (src/util/VNode.mjs:25 → :41 …)
```

A tab header button that no longer exists as a component is still referenced from the cockpit host's vnode tree, so every full-depth update of the host throws inside `VNode.createMap` and rejects (`VdomLifecycle` logs it as a fire-and-forget failure, rejects the parked promises, and re-diffs on the next cycle — against the same stale vnode). The reconciler's recovery (`settleFailedProjection`) needs a host update too and lands in its `unrecoverable` branch; the one scheduled re-projection fails the same way. From then on the projection chain is dead while the document keeps committing: the probe's final `activatePerspective('Overview')` **commits** (`[0.6078, 0.3922]` within 10s) in all five runs but the DOM never moves — the two shells stay exactly as they were. That is the hold: a staged shell that can never be un-hidden because the host can no longer be updated at all.

**Timing condition.** The Review switch has to enter the Focus refresh's settled tail immediately (the arm's `expect.poll` on the committed sizes resolves at commit time, before the Focus projection finishes — the refresh is scheduled `timeout(0)` after the commit). `neo-tab-header-button-13` is a header button the Focus commit built and retired; its component is gone before the Review commit's host update walks the tree that still holds it.

**Attribution: engine (dock projection), not the cockpit's projection wait.** The cockpit only commits documents and publishes from the settled hook; nothing in `apps/agentos` touches shells, header chrome or `hideMode`.

**Next.** Engine `dev` HEAD `57ae8880e4` carries neomjs/neo#18279 (merged today 14:32Z: "a refresh builds a stand-in only for an item of a new tabs node — header chrome instance-stable"), which is exactly the class of component the stale id names. Re-measuring the arm ×5 with that engine in `node_modules` (`--no-save`, the lock untouched) now; green → this ticket resolves through the next pin, red → an engine ticket with this receipt.

Probe: `test/playwright/e2e/agentos/FlipSettleProbeNL.spec.mjs` (untracked, kept out of the repo); logs in the origin session's scratchpad.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

- 2026-09-04T16:25:41Z @neo-fable-clio cross-referenced by #18297
### @neo-fable-clio - 2026-09-04T16:26:27Z

## Verdict — engine-side, filed as neomjs/neo#18297; this ticket stays open as the owner of the pin that carries the fix

- Engine `dev` HEAD `57ae8880e4` (with neomjs/neo#18279, the stand-in fix) does **not** close it: the arm ×5 → **4 red**, the probe (arm-exact timing) → never visible in 30 s in 3 of 4 runs, the same `Component not found for id: neo-tab-header-button-13` every time.
- The probe extended to every worker console type settles the recovery question: both failing flights (the host `neo-fm-fleet-cockpit-1`, then the staged shell `neo-container-98`) are logged by the fire-and-forget branch — the engine prints that line only when no promise is parked on the flight — and there is no `Dock projection failed` and no `Dock projection recovery failed` warning. So `reconcileProjection` does not reject into #18147's fail-safe; it hangs with the staged shell hidden, and every later commit queues behind the refresh tail.
- The engine ticket carries the reproduction (two topology commits back-to-back, the second inside the first's motion window), the root (a phase-2 silent retirement leaving its `{componentId}` in the host's next vnode) and the hang, with #18143's Fix-2 options as the decision the red-first arm now enables; offered to @neo-opus-vega by A2A, claimable by any peer.

AC-1 is met by the measurement above (attribution recorded). AC-2/AC-3 wait for the engine fix and the pin that carries it; the README's one stated red and the arm's hold comment stay until then — the hold comment's "settle" wording is corrected in the pin PR, not before.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

- 2026-09-04T17:33:03Z @neo-fable-clio cross-referenced by #107
- 2026-09-04T18:42:59Z @neo-opus-vega cross-referenced by PR #18321
- 2026-09-05T12:32:10Z @neo-fable-clio cross-referenced by #115
- 2026-09-12T10:04:58Z @neo-fable-clio cross-referenced by #123
- 2026-09-12T13:17:21Z @neo-fable-clio cross-referenced by #127
- 2026-09-12T21:01:34Z @neo-fable-clio cross-referenced by #133
- 2026-09-12T21:40:47Z @neo-fable-clio cross-referenced by PR #134
### @neo-fable-clio - 2026-09-12T21:41:48Z

Measurement at engine pin `7e8b32e421` (#133 / PR #134, 2026-09-12): the Review arm `FleetCockpitDockNL.spec.mjs:199` ran green headless on the Brain root — `.fm-agent-detail` visible after `activatePerspective('Review')` — with the Review duty now DECLARED as a center column (`review-split` → `detail-tabs`) instead of the pinned rail member (`items.detail.autoHidden = false`) this hold was measured on. One sample, not AC-2's 3/3 + headed.

What it attributes (AC-1, partial): the settle that never landed staged the RAIL member's tab chrome (`hideMode: 'visibility'`, un-hidden on the FLIP settle); a center member takes the ordinary tabs path and needs no un-hide. So the hold is specific to a rail member materializing pinned — which the cockpit's Review no longer does — and stays reachable through a snapshot restore that pins `detail` in the band (a capture taken while pinned) and through the rail reveal's pin action. Whether the engine's pipeline or the cockpit's wait owns the rail case is still unmeasured.

AC-2 is PR #134's post-merge item (3/3 headless + one headed run at the merged head); AC-3's README / spec-comment update follows that measurement.

Command: `NEO_AGENTOS_RUNTIME_ROOT=<brain checkout> NEO_E2E_PORT=8121 npx playwright test FleetCockpitDockNL -c test/playwright/playwright.config.e2e.mjs --workers=1` → 2 passed (6.7 s).

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session cf2ab186-e475-4418-9fe0-0352581faab3

### @neo-fable-clio - 2026-09-13T10:53:20Z

**Measurement sample at the #134 R1 head (`45f2b8d`, engine pin `7e8b32e421`), headless on the Brain root:** `FleetCockpitDockNL --repeat-each 3 --workers=1` → 6/6 — the Review arm (`:199`, the column route) 3/3, 14.6 s for the six; the settle hold fired in none of the three. Pre-merge sample only: AC-2's repeat at the merged head and the one headed run remain.

Run: `NEO_AGENTOS_RUNTIME_ROOT=<brain checkout> NEO_E2E_PORT=8147 npx playwright test FleetCockpitDockNL -c test/playwright/playwright.config.e2e.mjs --workers=1 --repeat-each 3`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session a1d12cdc-a975-4f69-a14e-50cc28343691

### @neo-fable-clio - 2026-09-13T11:11:14Z

**AC-2 measured at the merged head** (dev `2c2551bb8e`, PR #134 merged; engine pin `7e8b32e421`; Brain root `524b548`): `FleetCockpitDockNL` headless `--repeat-each 3 --workers=1` → 6/6, the Review arm (`:199`) 3/3, 14.4 s; then one headed run → 2/2, 7.7 s. The settle hold did not fire on the column route in any of the four Review runs. Together with the pre-merge sample above, that is 4/4 headless + 1 headed at this pin.

Remaining for the closeout: AC-3 (the README battery paragraph and the spec comment name the resolution — Review docks the inspector as a center column, #133 — instead of the hold) plus the deferred neo#18297 browser half (pane reachability and the shell at `shellIndex`, N=5) — one small PR next.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session a1d12cdc-a975-4f69-a14e-50cc28343691

- 2026-09-13T11:30:48Z @neo-fable-clio cross-referenced by PR #135
- 2026-09-13T12:23:59Z @tobiu referenced in commit `1ec040f` - "Merge pull request #135 from neomjs/agent/103-settle-closeout

docs(fleet): the battery's Review-preset red is closed, and the arm reads the shell and every committed pane (#103)"
- 2026-09-13T12:24:00Z @tobiu closed this issue

