---
id: 382
title: The cockpit shows no drop zones while a tab header is dragged
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-01T12:15:17Z'
updatedAt: '2026-10-01T18:14:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/382'
author: neo-fable
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
closedAt: '2026-10-01T18:14:43Z'
milestone: FM v1
---
# The cockpit shows no drop zones while a tab header is dragged

## Context

The operator, 2026-10-01 ~12:00Z, on the installed Fleet Manager (native Electron shell): a cockpit tab header can be dragged, but no drop zones appear. Captured first as a defect-note (A2A, 12:04Z), then measured in the served cockpit the same hour.

**Measured (served cockpit, Chromium headless, Institution dev `4f56f2a`, engine pin `e7d550e5dc`):** holding the `FLEET` tab header over the activity pane for 1.2 s produces a drag proxy (`neo-tab-header-toolbar … neo-dock-top`) and nothing else — zero `neo-dashboard-dock-drop-indicator*` elements at every 150 ms sample, the dock topology carries no indicators, and `Neo.manager.DragCoordinator`'s state reads `activeTargetZone: null`, `claimTrace: []`. Instance census in the AgentOS App Worker during the hold: `Neo.dashboard.dock.interaction.TabSortZone` 2, `Neo.dashboard.dock.window.Participation` 0, `Neo.dashboard.dock.interaction.DragAffordances` 0, `Neo.dashboard.dock.interaction.DropIndicators` 0. The shell symptom is therefore not Electron-specific and not a stale pin: the installed app (built 2026-09-30 21:46) carries engine `067f9fb93b`, which already contains PR #19254 and PR #19289.

## The Problem

The engine's drop feedback for a dock drag — the indicator menu (`DropIndicators`), the accept/reject preview (`Preview`) and the gesture controller that feeds them (`DragAffordances`) — is a consumer-composed tier. The Workstation composes it explicitly (`apps/workstation/view/Workspace.mjs:268-290` keeps `DockPreview` + `DockDropIndicators` as persistent siblings in a dedicated dock host; `apps/workstation/view/VesselWorkspace.mjs:255-275` composes `DragAffordances` over them and routes the projection's three cross-zone seams to it); the cockpit composes none of it. So in the FM a tab drag starts (the sort zones exist), nothing answers, and the user sees a header that follows the pointer into a layout that never responds. The dock-layouts half of FM v1 is unreachable from the product while this holds.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/Container.mjs` sets `enableDockTearOutLifecycle: true` and projected its shell straight into the root vbox after the control bar (`dockShellIndex: 1`); `apps/agentos/view/fleet/cockpit/VesselContainer.mjs:33` extends `Neo.dashboard.dock.Workspace`. Neither file referenced `DragAffordances`, `Preview` or `DropIndicators`.
- Engine record (`src/dashboard/dock/Workspace.mjs` class doc): the façade "owns no concern outright … declinable stateful affordances, the tear-out gesture machine, multi-window lifecycle — belongs to a named owner beside it"; the projection mounts into "a dedicated child named by `dockHostReference` when a consumer keeps persistent overlay siblings (a preview renderer, drop indicators) beside the projected shell"; `getDockProjectionOptions` carries "a drag-affordance layer's `onDockCrossZoneDragMove` / `onDockCrossZoneDragCancel` / `onDockCrossZoneDrop` seams". `enableDockTearOutLifecycle` composes `TearOut` (`:622`), not the affordance tier. The default is inert by design.
- `src/dashboard/dock/interaction/DragAffordances.mjs` class doc: "the app-neutral drag-affordance gesture controller every docking workspace composes" — owner duck-type = `dockModel` + `applyDockZoneOperation` + `onDockZoneDocumentChange`; the composer supplies `host`, `preview`, `indicators`.
- The cross-window seat (`src/dashboard/dock/window/Participation.mjs`) is a different concern: measured at `fm-cockpit-dock` with a registered participant, the coordinator's claim trace reads `outcome: 'no-claim'`, the only candidate `skipped: 'source-or-excluded'` — the coordinator never targets the source window, so a participant alone changes nothing for an in-window drag. Out of scope here.
- Skin contract (`resources/scss/src/apps/workstation/Workspace.scss:291-310`): the host owes the overlays exactly `position: relative`; the engine owns their inset / position / z-index. `src/dashboard/dock/interaction/DropIndicators.mjs:137,171,338`: the chips exist at rest, the layer hides by `neo-dashboard-dock-drop-indicators-hidden`, the hovered chip carries `…-indicator-active`.

## The Fix

The Workstation's shape, in the cockpit:

1. `Container.mjs` declares a dock host under the control bar (`reference: 'dock-host'`, `cls: ['fm-cockpit-dock-host', 'neo-dashboard']`, fit layout) holding the two persistent overlays (`DockPreview` as `dock-preview`, `DockDropIndicators` as `drop-indicators`); `dockHostReference: 'dock-host'`; the projected shell mounts at the host's first slot (`getDockHost().insert(dockShellIndex, projectDockModel())`; the engine default `dockShellIndex: 0`); the duplicated `dockShellIndex` / `dockProjectionConfig` pair goes.
2. `VesselContainer.mjs` composes `DragAffordances` over the host and its overlays in `onConstructed`, routes `onDockCrossZoneDragCancel` / `onDockCrossZoneDragMove` / `onDockCrossZoneDrop` to it in `getDockProjectionOptions()`, and retires it in `destroy`.
3. One cockpit skin rule: `.fm-cockpit-dock-host { position: relative }`.

No engine change; no new module; the probe becomes the tracked red-first e2e arm.

## Contract Ledger Matrix

| Target surface | Source of authority | Today | Proposed | Fallback | Docs | Evidence |
|---|---|---|---|---|---|---|
| `AgentOS.view.fleet.cockpit.Container` items + `dockHostReference` (`apps/agentos/view/fleet/cockpit/Container.mjs`) | `Neo.dashboard.dock.Workspace` façade contract (class doc, `dockHostReference`) | shell projected into the root vbox at index 1; no overlays | a declared dock host under the bar holds the shell (slot 0) and the two persistent overlays | engine refuses a `dockHostReference` that resolves to no live host (loud) | Container.mjs items doc | AC-2, AC-3 |
| `AgentOS.view.fleet.cockpit.VesselContainer#dragAffordances` + `getDockProjectionOptions` seams | `DragAffordances` class doc; Workspace `getDockProjectionOptions` hook | no gesture controller; the projection's default cross-zone seams | the controller answers every in-window drag with the indicator menu + preview and commits the release through `applyDockZoneOperation` / `onDockZoneDocumentChange` | — | VesselContainer class doc | AC-1, AC-2 |
| `.fm-cockpit-dock-host` (cockpit `Container.scss`) | Workstation host skin precedent | — | `position: relative` — the overlays' containing block | — | the rule's comment | AC-3 |
| `Neo.dashboard.dock.interaction.DragAffordances`, `Preview`, `DropIndicators` | engine, unchanged | consumer-composed | unchanged | — | — | AC-1 |

## Decision Record impact

`aligned-with` ADR 0029 (docking design: the façade / named-owner placement; the affordance tier stays consumer-composed). Engine follow-up to raise with the record's author, not done here: whether a workspace with `enableDockTearOutLifecycle` should compose the in-window affordance tier itself (three consumers compose it by hand today).

## Acceptance Criteria

- [ ] AC-1 Red-first e2e arm (`FleetCockpitTabDragIndicatorsNL`): a tab header held over the other cockpit pane un-hides the indicator menu within 2 s, exactly one chip is `…-indicator-active`, and the menu's `activeCandidate` is non-null in the App Worker; red on dev (the menu never mounts), green at the head.
- [ ] AC-2 Releasing on the active chip commits through the cockpit's reducer: the dragged item leaves its zone in the committed document and every other item stays, the menu hides with no chip active, and the cockpit's existing dock witnesses (`FleetCockpitDockNL`, `FleetCockpitPopOutNL`, `FleetCockpitNWindowNL`, the rail witnesses) stay green. (`getDockTopology().operations` is the executable vocabulary, not a history — no count clause.)
- [ ] AC-3 Visual tier: idle cockpit goldens unchanged in both skins (the host is a fit-layout wrapper at the shell's old flex slot; the overlays render nothing at rest).
- [ ] AC-4 `[L3-deferred — operator handoff needed]` Post-merge, installed receipt: the operator drags a tab header in the installed Fleet Manager after the next repackage and sees the zones (his observation is the acceptance). This ticket closes with PR #383, so the receipt's surviving owner is Institution #12 (the native-shell acceptance owner); recorded there when it lands *(re-pointed 2026-10-01 by the author on @neo-gpt-emmy's RA-1)*.

## Out of Scope

The cross-window seat (`Participation`) for the cockpit window — measured inert for in-window drags, and the cockpit's vessels are shells, not dock workspaces; the engine default for the tier (named above as a follow-up with the ADR 0029 author); drop-commit animation and the design polish of the indicator menu (Clio's 2026-07-10 exploration, Memory Core).

## Avoided Traps

- Treating the shell as the culprit: the served cockpit shows the same absence headless; the Electron window uses a default frame (no drag regions), and the installed pin already carries the park work.
- Seating a `Participation` as the fix: it registers and the coordinator still never claims the source window (`skipped: 'source-or-excluded'`, measured 12:4xZ) — the in-window tier is the gesture controller's, and the ticket's first prescription said otherwise until the measurement corrected it.
- Adding `DropIndicators` as a bare child: without a `DragAffordances` fed by the projection's cross-zone seams the menu never receives a candidate set — the Workstation's wiring shows the three parts travel together, over a positioned host.

## Related

- Institution #12 (native shell UX), #11 (visual baseline harness); the FM v1 milestone (dock layouts are in its scope per the operator, 2026-10-01).
- Engine: PR #19254 / PR #19289 (the park work the installed pin carries), neo #18151 (the dock asks consumers to re-derive engine knowledge — closed, the same tension).

Live latest-open sweep: latest 20 open Institution issues at 2026-10-01 12:14Z, no equivalent; A2A latest 30 (all read states) at 12:14Z carried no competing claim; Memory Core sweep ("cockpit tab header drag no drop indicators Participation") returned the 2026-07-10 exploration that the affordances exist in the engine example and no decision that the cockpit should lack them; own-assignment sweep: #9 (epic) only. Structure map: N/A (no new file).

Origin Session ID: b04b2ce2-c8b8-40ba-9f0a-4dd2585a5be1 (rotated mid-lane to 771833a0-8bd1-4520-8e2c-138d82c15a53; the PR #383 turn is logged under the second)
Retrieval Hint: "cockpit tab header drag no drop indicators DragAffordances dock host"



## Timeline

- 2026-10-01T12:15:18Z @neo-fable assigned to @neo-fable
- 2026-10-01T12:15:19Z @neo-fable added the `bug` label
- 2026-10-01T12:15:19Z @neo-fable added the `agent-os` label
- 2026-10-01T12:15:19Z @neo-fable added the `ai` label
- 2026-10-01T12:16:16Z @neo-fable added this to the **FM v1** milestone
- 2026-10-01T13:04:05Z @neo-fable cross-referenced by PR #383
- 2026-10-01T13:38:29Z @neo-fable-clio cross-referenced by #384
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392
- 2026-10-01T16:05:38Z @neo-fable referenced in commit `4915e3d` - "chore(merge): bring origin/dev into the branch (#382)

# Conflicts:
#	test/playwright/visual/__screenshots__/baseline-inputs.txt"
- 2026-10-01T16:15:13Z @neo-fable cross-referenced by #19350
- 2026-10-01T17:55:03Z @neo-fable cross-referenced by #12
- 2026-10-01T17:56:27Z @neo-fable referenced in commit `cc65abd` - "test(cockpit): the drop-indicator arm describes the composed in-window controller (#382)

Review round 1: the spec's header and AC-1 comment credited a Participation and the coordinator — the stale first prescription. The tier is the cockpit's composed DragAffordances over the declared dock host; the coordinator never targets an in-window drag's own window."
- 2026-10-01T18:14:43Z @tobiu referenced in commit `8b730f5` - "feat(agentos): the cockpit answers a dragged tab header with drop zones (#382) (#383)

* feat(agentos): the cockpit answers a dragged tab header with drop zones (#382)

A tab header dragged inside the Fleet cockpit followed the pointer into a layout that never
answered: the engine's in-window drop feedback (the preview renderer, the drop-indicator menu
and the gesture controller that feeds them) is a consumer-composed tier, and the cockpit
composed none of it — measured headless as zero indicator chips and no DragAffordances,
DropIndicators or Participation instance in the App Worker, the same absence the operator saw
in the installed Electron shell.

The cockpit now declares a dock host under its control bar (`dockHostReference`), holding the
projected shell at slot 0 beside the two persistent overlays; VesselContainer composes
DragAffordances over them and routes the projection's three cross-zone drag seams to it; one
skin rule makes the host the overlays' containing block. The duplicated shell-index pair in
the cockpit config goes with it. A held header now un-hides the indicator menu with the hovered
chip active, and releasing on it commits through the cockpit's own reducer.

Red-first e2e arm (FleetCockpitTabDragIndicatorsNL); the shell-placement assertions in
FleetCockpitDockNL and the projection unit spec move to the dock host's first slot.

* test(cockpit): the drop-indicator arm describes the composed in-window controller (#382)

Review round 1: the spec's header and AC-1 comment credited a Participation and the coordinator — the stale first prescription. The tier is the cockpit's composed DragAffordances over the declared dock host; the coordinator never targets an in-window drag's own window."
- 2026-10-01T18:14:44Z @tobiu closed this issue

