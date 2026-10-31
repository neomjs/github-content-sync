---
id: 846
title: 'The installed FM lacks Brain #1355, so Antigravity seats open at the centre'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees: []
createdAt: '2026-10-30T23:02:10Z'
updatedAt: '2026-10-31T00:48:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/846'
author: neo-opus-ada
commentsCount: 1
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
# The installed FM lacks Brain #1355, so Antigravity seats open at the centre

## Context

Operator report, 2026-10-30 ~23:0xZ: Claude and Codex seat windows return to their previous positions on the second monitor, but Antigravity (Gemini) seats still open at the laptop's centre. The fix was meant to arrive with Brain #1355 (merged 2026-10-29, permission-free placement through CDP).

## The Problem

The FM places seat windows from **its own runtime**, not from the plane or deploy home. Tonight's installed candidate carries Brain `05e4285` (2026-10-28), which **predates #1355**: `placeDevToolsWindowRect` occurs 0 times in the FM runtime's `seatWindowRect.mjs` and 3 times in Brain `dev`. The FM therefore still places Antigravity through System Events, and macOS refuses it: `window restore refused: osascript is not allowed assistive access (-25211)`, logged at 22:44:30Z and 22:56:11Z. An FM reinstall with a new ad-hoc signature also drops any Accessibility grant the old bundle held.

## The Fix

Ship an FM build whose organism carries Brain ≥ `5e749c85` (#1355). Build from a Brain checkout at current `dev`, rehearse it, install it (I may install per the operator's 10-30 ruling), then confirm an Antigravity seat opens at its recorded rectangle. Until then, granting Accessibility to the newly installed `Neo Harness.app` is the operator's stop-gap (System Settings → Privacy & Security → Accessibility).

## Acceptance Criteria

- [ ] AC-1 The installed FM runtime's `seatWindowRect.mjs` contains #1355's CDP placement (`organism-build-info.json` Brain revision ≥ `5e749c85`).
- [ ] AC-2 An Antigravity seat started from the installed FM opens at its recorded rectangle on the second monitor with no Accessibility grant (operator receipt).

## Related

Brain #1351/#1355, the drag ticket filed beside this one (Brain), #424/#516 (tonight's H4 installed candidate `a4420fd` / Brain `05e4285`).

Sweep: searched open/closed "antigravity window" in Institution and Brain at 2026-10-30T23:0xZ; no equivalent.

unowned-rationale: The next FM build carries it; Ada takes it if unclaimed (FM installs are hers per the operator's ruling).

Retrieval Hint: "installed FM runtime predates Brain 1355 Antigravity placement TCC -25211"

Origin Session ID: 2e07d6d3-af03-4b39-9254-f979cc291d2f

## Timeline

- 2026-10-30T23:02:12Z @neo-opus-ada added the `bug` label
- 2026-10-30T23:02:12Z @neo-opus-ada added the `agent-os` label
- 2026-10-30T23:02:12Z @neo-opus-ada added the `ai` label
### @neo-opus-ada - 2026-10-31T00:48:54Z

**AC-1 receipt: installed 2026-10-31T00:47Z.**

The installed FM's `organism-build-info.json` names Brain `4ca0f506` (the plane's deployed revision), which descends from #1355's merge `5e749c85`.

The installed runtime's `ai/services/fleet/seatWindowRect.mjs` (generation `f4e5c7e00b44…`) is byte-identical to `5e749c85`'s: sha1 prefix `f79e3d3f5f10` for both.

AC-2 still needs your receipt: the next Antigravity seat started from the FM should open at its recorded rectangle on the second monitor, with no Accessibility grant.

⚖️ Ada



