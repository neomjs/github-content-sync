---
id: 922
title: The FM window takes 15–27 s to appear after launch
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees: []
createdAt: '2026-11-11T18:40:12Z'
updatedAt: '2026-11-11T18:40:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/922'
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
---
# The FM window takes 15–27 s to appear after launch

## Context

The operator reported this on 2026-11-11, on FM Institution `6f00d3da` / Brain `7219f4e1` / Engine `c7e19744`: "gut feeling 10-15s before the electron shell shows." I measured it on three launches during row 5's walk.

**Sweeps, 18:38:51Z:**
- **Live latest-open + closed:** 54 open, red band. Searched `launch window appears`, `cold launch seconds`, `window delay` and `startup time`. #12 (the native shell spec) is the broad owner of the "download and run" moment, and no ticket covers this delay.
- **A2A:** the 12 newest messages. No claim.
- **MC:** no prior decision.

## The Problem

| Launch | Process started | First `main.log` line (`HARNESS_SEAT_MOVE_OUTCOME`) | App page connected | Silent gap |
|---|---|---|---|---|
| Install relaunch, `install.mjs --open` | 17:37:32Z (installer end) | 17:37:59Z | 17:37:59Z | **27 s** |
| Operator quit + reopen | quit at 17:52:43Z; click time unknown | 17:53:10Z | 17:53:10Z | operator: 10–15 s |
| Operator reopen (H3) | 18:15:09Z (`ps` elapsed) | 18:15:24.356Z | 18:15:24.476Z | **15 s** |

Nothing is logged until `HARNESS_SEAT_MOVE_OUTCOME {"state":"committed","retirement":{"state":"retired","archived":[]}}` and `HARNESS_SEAT_ROOT {"origin":"moved"}`. The window and the app page come up right after.

**Unknown:** what runs in those 15–27 s. Electron start, the seat-root move, and runtime checks all happen before the first log line.

## The Architectural Reality

- **Design authority:** #12, the native shell's "download and run" moment, which owns first-launch expectations; it sets no number. Expected: a window within a couple of seconds of launch, with the boot showing its state inside it.
- **The main process** is `harness/main.mjs`. Its seat-move/seat-root steps log first, and the boot plan follows (`HARNESS_BRAIN_PLAN`).

## The Fix

1. Add timestamps from process start to the window: app ready, each pre-window step, window created, page loaded. Name the step that takes the time.
2. Show the window before the slow step, with its progress on the spine ("Fleet starting"), or make the step fast.

## Acceptance Criteria

- [ ] `main.log` shows the pre-window steps with timings on a launch of the installed FM. One receipt names the slow step.
- [ ] After the fix, the installed FM's window appears within 3 s of launch on this machine. One receipt.

## Out of Scope

- Time to `live` after the window shows: the activity feed's lag and #917.

## Related

- #12 and #424.
- `#732`: an FM update never requires closing a harness.

unowned-rationale: The measurement step is ready for a peer; maintainers provide review.

Origin Session ID: b261f993-5994-4a53-bb9c-9fac974a9029
Retrieval Hint: "FM window appears 15-27 seconds after launch silent main.log"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-11T18:40:13Z @neo-opus-ada added the `bug` label
- 2026-11-11T18:40:13Z @neo-opus-ada added the `ai` label
- 2026-11-11T18:40:13Z @neo-opus-ada added the `performance` label

