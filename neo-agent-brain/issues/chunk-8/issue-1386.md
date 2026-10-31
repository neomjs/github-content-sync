---
id: 1386
title: 'Electron windows cannot be dragged on macOS 27.2 (Antigravity, Claude, Codex)'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T23:02:08Z'
updatedAt: '2026-10-31T02:03:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1386'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-10-31T02:03:11Z'
---
# Electron windows cannot be dragged on macOS 27.2 (Antigravity, Claude, Codex)

## Context

Operator report, 2026-10-30 ~23:0xZ: a Gemini seat started from the FM opens its Antigravity window at the laptop's centre, and **the window cannot be dragged with the mouse; resizing works**. The operator called it a killer bug.

## Observations (read-only, 23:0xZ)

- Seat `neo-gemini-maryam`: Antigravity 2.28.2 runs with `--user-data-dir=<seat>/harness/antigravity`. Its language server runs with `--subclient_type hub`. Its one DevTools target is a page titled "Antigravity" at `https://127.0.0.1:<port>/?section=<uuid>`, i.e. the Agent Manager, not the editor workbench.
- The page has loaded (`readyState: complete`) and **declares 6 `app-region: drag` elements**, so the drag regions exist in CSS.
- Nothing is attached to the window's DevTools port: `lsof` shows no client besides a probe that had already closed. The FM is not holding a debugger session.
- **Antigravity self-updated on 2026-10-29 at 06:41 (local) to 2.28.2.** Between the previous installed FM (Brain `1bf8bd91`) and tonight's candidate (Brain `05e4285`), `ai/services/fleet` changed only `fleetGoldenPathSource.mjs` and `seatMemoryLayerTemplate.mjs`. Institution `20d89be..a4420fd` touches no seat-launch or window code. Our launch path is unchanged.
- Placement, a separate cause: the FM log reads `window restore refused: osascript is not allowed assistive access (-25211)` for `neo-gemini-julia` (22:44:30Z) and `neo-gemini-maryam` (22:56:11Z). The installed FM runtime predates #1355's permission-free CDP placement (see the Institution ticket filed beside this one).

## Settled (2026-10-30, 23:1xZ)

- **Not ours.** An Antigravity window opened by hand, with no FM and no seat profile, cannot be dragged either (operator, 23:0xZ).
- **No roaming process.** No window manager is running (yabai, Rectangle, Magnet, Hammerspoon, BetterTouchTool, skhd, AeroSpace), and neither is any placement script or osascript/System Events loop. The hand-opened instance (pid 28061) listens on its DevTools ports (2.28.2 writes `DevToolsActivePort` by default), but no client is connected.
- **Two updates on the same day:** Antigravity self-updated to **2.28.2 (Electron 44.5.1)** on 2026-10-29 at 06:41 local, and **macOS updated to 27.2** at 13:19 CET. Claude runs Electron 44.9.3; whether its windows still drag is the next discriminator between "Electron 44.5 on macOS 27.2" and "Antigravity 2.28.2".
- Operator workaround that bypasses the app's drag regions: `defaults write -g NSWindowShouldDragOnGesture -bool true`, then relaunch the app. Ctrl+Cmd+drag then moves any window. This is a user-level macOS setting, so the operator runs it.
- Seat windows can still be placed into their recorded slot through CDP, without Accessibility, which is #1355's mechanism. Inst #846 brings that to the installed FM.

**Disposition:** an upstream defect (Antigravity, or Electron 44.5 on macOS 27.2). Our part: keep seat placement working (Inst #846) and recheck after Antigravity's next update. AC-1 then becomes an upstream fix receipt.

## The Fix

Settle the open question, then choose:
- **(a) Upstream:** launch seats into the editor window, or a window type that drags, and record the upstream defect.
- **(b) Ours:** correct the seat launch/profile so the window is draggable.

The operator's immediate workaround in either case: move the window through the green traffic-light button's "Move to <display>" or the Window menu's move commands, which don't depend on drag regions.

## Acceptance Criteria

- [ ] AC-1 An FM-started Antigravity seat window can be dragged with the mouse on the installed FM (operator receipt).
- [ ] AC-2 The cause is recorded: upstream (version + repro) or ours (the commit).
- [ ] AC-3 A seat launch spec or receipt pins the window type the FM opens.

## Related

#1351, #1355 (placement via CDP), #1301 (TCC -25211 surfaced), Institution #843–#845 (tonight's walk).

Sweep: searched open/closed "antigravity window" in Brain and Institution plus the latest open issues at 2026-10-30T23:0xZ; no equivalent (#1351/#1301/#1077 are placement, all closed).

Retrieval Hint: "Antigravity seat window cannot be dragged agent manager hub 2.28.2"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f

## Timeline

- 2026-10-30T23:02:08Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T23:02:09Z @neo-opus-ada added the `bug` label
- 2026-10-30T23:02:10Z @neo-opus-ada added the `ai` label
- 2026-10-30T23:02:10Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T23:18:17Z @neo-opus-ada changed title from **An FM-started Antigravity seat window cannot be dragged (resizing works)** to **Electron windows cannot be dragged on macOS 27.2 (Antigravity, Claude, Codex)**
### @neo-opus-ada - 2026-10-30T23:18:18Z

**Scope widened (operator, 23:1xZ):** Claude and Codex harness windows cannot be dragged either, so this is not Antigravity-specific.

**Upstream match:** [anthropics/claude-code#102613](https://github.com/anthropics/claude-code/issues/102613) reports, on macOS 27.2 (beta 4, Electron 44.7.2), that title-bar dragging fails in Electron and Chromium apps with custom chrome (`performWindowDragWithEvent:`). AppKit apps are unaffected. It often starts after remote or virtual input, and a WindowServer restart clears it. Our own wake plane injects synthetic input into seat windows (osascript adapters), so the trigger may recur here.

**Mitigations:** a clean logout or restart (it restarts WindowServer; `killall WindowServer` force-quits the whole session, including Docker Desktop and the plane). `NSWindowShouldDragOnGesture` (Ctrl+Cmd+drag) works regardless.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

### @neo-opus-ada - 2026-10-31T02:03:10Z

**Resolved by a macOS restart. The cause was session state, not Antigravity and not us.**

- **AC-1 (operator receipt, 2026-10-31):** after the system restart (boot 2026-10-30T23:27:51Z), window movement works again.
- **AC-2 (cause):** the versions are the same ones that failed. macOS is still 27.2, Antigravity still 2.28.2 (installed 10-29 06:41 local), and its Electron still 44.5.1. Only the session changed, so the "upstream defect" disposition above is withdrawn. The likeliest reading is a stuck window-server state after the 27.2 update.
- **AC-3 (pin the window type):** retired. It served Fix (a), and the window type was not the cause.

Still-valid intent: seat placement without Accessibility. It lives in Institution #846; AC-1 was met tonight, and AC-2 is waiting on the operator's next Antigravity seat start.

The `NSWindowShouldDragOnGesture` workaround has been reverted (`defaults delete -g`, verified gone).

⚖️ Ada

- 2026-10-31T02:03:11Z @neo-opus-ada closed this issue

