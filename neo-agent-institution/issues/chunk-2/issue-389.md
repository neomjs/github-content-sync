---
id: 389
title: Detail and memories panes duplicate the dock header's pop-out verb
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-10-01T14:18:14Z'
updatedAt: '2026-10-01T15:12:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/389'
author: neo-fable-clio
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
closedAt: '2026-10-01T15:12:54Z'
milestone: FM v1
---
# Detail and memories panes duplicate the dock header's pop-out verb

## Context

Operator observation on the installed Fleet Manager (2026-10-01, Agent Detail for `neo-gpt-sophie`): the dock header of the Agent Detail pane carries the layout's own window actions — lock · refresh · mute · **"Pop out into a window"** · maximize · close (the tooltip is visible in the capture) — and the pane's body shows a second pop-out icon at the trailing edge of the Status / Configuration tab strip. The second one predates dock layouts getting header actions; it is the shell-era verb.

## The Problem

Two controls for one verb on one pane. The tab-strip icon is the older placement (T4.15's "shell-supplied window verbs ride the tab header bar's ACTION seam"), written before the dock header owned window verbs; now it duplicates the header action and sits where a content action would be expected (next to the tab labels), so it reads as a tab-level link rather than a window verb.

## The Architectural Reality

- `apps/agentos/view/fleet/cockpit/Container.mjs:890-895` passes `shellTools: [me.buildDetailWindowToggle()]` into the detail pane; `:922` passes `shellTools: [me.buildMemoriesWindowToggle()]` into the memories pane.
- `apps/agentos/view/fleet/detail/Container.mjs:290-295` places `shellTools` on `detail-tabs.headerActions` (the "ACTION seam"); `apps/agentos/view/fleet/memories/Container.mjs:57-59, :230` adds them to `memories-actions`.
- The dock header's own actions (incl. pop-out) come from the engine's dock layout (`Neo.dashboard.dock.Workspace`, `enableDockPopOutAction` double-gated on `enableDockTearOutLifecycle`) — the layout owns the verb now; `VesselContainer.mjs:124-144` documents the `shellTools` slot as the earlier layout-blind carrier.

## The Fix

*(Restated 2026-10-01 ~14:30Z after the source read — the pane-side verb is not only a pop-out: in the vessel it is the pane's way home, and four e2e arms plus the unit chrome spec drive the round trip through it.)* Keep the layout's verb for the docked state and the pane's verb for the away state, never both at once: the two builders in `VesselContainer.mjs` render the toggle `hidden` (`removeDom`) by default as a **return** verb ("Return detail" / "Return memories", down-left glyph); `syncVesselChrome` shows it only while the pane is vessel-owned or in flight (disabled while in flight), so the docked pane's tab strip carries no icon and the dock header's "Pop out into a window" is the single pop-out. The three e2e arms that detached through the pane toggle (`FleetCockpitPopOutNL`, `FleetPermanenceMatrixRow4NL`, `FleetCockpitDrillRoundTripNL`) detach through the dock header's own action instead, and `FleetCockpitPopOutNL` asserts the docked strip shows no toggle; the unit chrome spec asserts hidden-while-docked / shown-while-away. The `shellTools` slot, the recall chrome in the bar and the SCSS stay (they serve the away state). Goldens of the detail and memories panes re-captured from a full visual run (build-themes → `test-visual` alone → stamp → commit).

## Acceptance Criteria

- [ ] AC-1 Docked, the Agent Detail's tab strip shows no trailing icon (`.fm-detail-window-toggle` count 0, `removeDom`) and "Pop out into a window" exists in the dock header. E2e assertion in `FleetCockpitPopOutNL` + visual golden.
- [ ] AC-2 Docked, the memories pane's actions row carries only its own actions. Visual golden.
- [ ] AC-3 The round trip still works through the dock header's admission: the three e2e arms detach via the engine's pop-out pathway, the pane in the vessel shows the return verb ("Return detail" / "Return memories") and returns through it; the unit chrome spec asserts hidden-while-docked, disabled-while-in-flight, shown-while-away.
- [ ] AC-4 The bar's recall chrome is unchanged (visible only while a pane is away).

## Out of Scope

The dock header's action set itself (engine); the detail panes' empty sources (the sibling ticket filed with this one); #386/#387's native title bar.

## Related

#7 (shell lane; the T4.15 pop-out leaf that introduced the slot) · #13 (design conformance) · #24 (cockpit view layer) · the sibling ticket "The Agent Detail's four status panes have no live source" · neo#15239 (dock tear-out epic, the layout's verb)

Decision Record impact: aligned-with ADR 0029 (the layout owns the window verbs; consumers do not re-add them).

Sweeps: live latest-open sweep — the latest 20 open Institution issues at 2026-10-01T14:17:25Z (newest #388), no equivalent; A2A in-flight sweep — last 8 messages at 14:17Z, no claim on the detail/memories panes (claims: #386/#387/#388 shell, #381 pin); Memory Core rationale sweep — no prior decision surfaced (index noise); own-assignment sweep — #24 adjacent (same view layer), none on this; structure map — Brain-hosted, N/A; owning files cited above.

Origin Session ID: 6682a116-897e-4c18-925e-4320d0489481

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 6682a116-897e-4c18-925e-4320d0489481


## Timeline

- 2026-10-01T14:18:14Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-01T14:18:16Z @neo-fable-clio added the `bug` label
- 2026-10-01T14:18:16Z @neo-fable-clio added the `agent-os` label
- 2026-10-01T14:18:16Z @neo-fable-clio added the `ai` label
- 2026-10-01T14:18:16Z @neo-fable-clio added the `design` label
- 2026-10-01T14:19:10Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392
- 2026-10-01T14:36:19Z @neo-fable-clio cross-referenced by PR #394
- 2026-10-01T15:12:54Z @tobiu referenced in commit `8e08f46` - "fix(cockpit): the pane-side window verb shows only while its pane is away; docked, the dock header owns pop-out (#389) (#394)"
- 2026-10-01T15:12:54Z @tobiu closed this issue

