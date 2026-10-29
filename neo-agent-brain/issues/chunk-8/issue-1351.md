---
id: 1351
title: 'FleetLifecycleService: restore Antigravity seat window placement on second monitor without macOS Accessibility TCC'
state: OPEN
labels:
  - bug
  - ai
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-29T13:54:32Z'
updatedAt: '2026-10-29T13:54:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1351'
author: neo-gemini-julia
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
# FleetLifecycleService: restore Antigravity seat window placement on second monitor without macOS Accessibility TCC

## Problem / Goal

Starting Fleet Manager seats on macOS previously placed GUI harnesses back at their saved positions and sizes on the secondary monitor (`<instanceHome>/.neo-fleet-window.json`). While **Claude Desktop** (`<instanceHome>/window-state.json`) and **Codex Desktop** (`<instanceHome>/codex-home/.codex-global-state.json` → `electron-main-window-bounds`) continue to open on the secondary monitor because their Electron shells persist window geometry inside their isolated profile directories, **Antigravity (`Antigravity.app`)** regressed to opening centered on the primary laptop display at its hardcoded default `1400×900` frame (`dist/utils.js:1301`: `new BrowserWindow({ width: 1400, height: 900, minWidth: 500, minHeight: 400 })`).

In addition, every `Stop` (`captureSeatWindow`) and `Start` (`restoreSeatWindow`) in `FleetLifecycleService` currently fails in `~/Library/Logs/Neo Harness/main.log` with:

```text
HARNESS_BRAIN [FleetLifecycleService] seat '<id>' window restore refused: osascript is not allowed assistive access (-25211) (grant Accessibility permission to Neo Harness.app in System Settings).
```

and triggers macOS's `universalAccessAuthWarn` prompt.

## Root Cause & Empirical Findings

1. **False TCC identity assumption in `ai/services/fleet/seatWindowRect.mjs`:**
   - `seatWindowRect.mjs` assumed `FleetLifecycleService` runs under the same process and `kTCCServiceAccessibility` grant as the wake receiver (`com.neomjs.agent-os-wake.plist` → `/opt/homebrew/bin/node`).
   - In reality, `FleetLifecycleService` runs inside the packaged `Neo Harness.app` brain daemon spawned from a per-generation runtime copy (`~/Library/Application Support/neo-harness/brain/runtime/<sha>/host.runtime/Contents/MacOS/Neo Harness`) built unsigned (`mac.identity: null`). Every runtime update changes the ad-hoc `cdhash` and executable path, invalidating any `kTCCServiceAccessibility` grant on `System Events` window queries (`-25211`).
2. **Zero-TCC window bounds inspection via CoreGraphics (`CGWindowListCopyWindowInfo`):**
   - `osascript -l JavaScript` importing `CoreGraphics` and calling `$.CGWindowListCopyWindowInfo(1 | 16, 0)` (`kCGWindowListOptionOnScreenOnly | kCGWindowListExcludeDesktopElements`, `kCGNullWindowID`) returns `kCGWindowBounds` (`{X, Y, Width, Height}`) in global screen points for any `kCGWindowOwnerPID` and `kCGWindowLayer === 0` in `~49ms` with **zero TCC permissions** (neither Accessibility `-25211`, nor Automation `-1743`, nor Screen Recording) and never spawns `universalAccessAuthWarn`.
   - Selecting the largest positive-sized layer-0 window for `kCGWindowOwnerPID === pid` accurately resolves the main window across Antigravity, Claude Desktop, and Codex Desktop (even when an app has a smaller auxiliary window).
   - With zero-TCC `readSeatWindowRect`:
     - `captureSeatWindow` on `Stop` succeeds for **all** GUI seats without Accessibility.
     - `waitForSeatWindowRect` on `Start` succeeds for **all** GUI seats without Accessibility, allowing Claude Desktop and Codex Desktop seats that self-restore their profile bounds to immediately return `state: 'kept'` without touching `System Events`.
3. **Zero-TCC cross-display window placement for Antigravity via profile-local `DevToolsActivePort` CDP:**
   - `Antigravity.app` always launches with `--remote-debugging-port=0` (`dist/main.js:758`) and writes `<instanceHome>/DevToolsActivePort` (`<port>\n/devtools/browser/<id>`).
   - Connecting to `ws://127.0.0.1:${port}${browserPath}`, waiting for a `page` target with a loopback `http(s)` origin, calling `Browser.grantPermissions({ permissions: ['windowManagement'], origin })` (omitting `browserContextId` for Electron's default session so Blink does not clamp `window.moveTo` to the primary display), attaching via `Target.attachToTarget({ targetId, flatten: true })`, and evaluating `window.moveTo(x, y); window.resizeTo(width, height); window.moveTo(x, y);` with `userGesture: true` moves and resizes the Antigravity window across monitors in one step with zero macOS TCC permissions.

## Architectural Direction

- **File placement (`structural-pre-flight` Stage 1 fast-path):** Modify existing `ai/services/fleet/seatWindowRect.mjs` and `ai/services/fleet/FleetLifecycleService.mjs` (plus their unit test suites `test/playwright/unit/ai/services/fleet/seatWindowRect.spec.mjs` and `test/playwright/unit/ai/FleetLifecycleService.spec.mjs`). No new `.mjs` files needed.
- In `ai/services/fleet/seatWindowRect.mjs`:
  1. Replace `System Events` in `readWindowRectArgs(pid)` with a zero-TCC JXA `CoreGraphics.CGWindowListCopyWindowInfo(1 | 16, 0)` query filtering `kCGWindowOwnerPID === pid && kCGWindowLayer === 0` and picking the largest positive-sized window.
  2. Add `readDevToolsActivePort(instanceHome, { fileSystem })` and `placeDevToolsWindowRect(rect, { instanceHome, timeoutMs, fileSystem, WebSocketCtor, sleep })` to place the window over loopback CDP using `Browser.grantPermissions({ permissions: ['windowManagement'], origin })` + `Runtime.evaluate` (`userGesture: true`).
  3. Update `placeSeatWindow(pid, rect, options)` to attempt `placeDevToolsWindowRect` first when `options.instanceHome` provides a `DevToolsActivePort` (or `options.placeDevToolsFn`), then verify/poll the settled frame via `readSeatWindowRect`, and only fall back to `osascript` `placeWindowRectArgs(pid, rect)` when CDP placement is unavailable or does not reach the target rect.
- In `ai/services/fleet/FleetLifecycleService.mjs`:
  1. Update `getWindowRectPlaceFn()` to forward `options` to `placeSeatWindow(pid, rect, options)`.
  2. Update `restoreSeatWindow(record, { now })` to pass `{ instanceHome: record.instanceHome, harnessType: record.harnessType }` into `place(...)`.

## Acceptance Criteria

- [ ] `readWindowRectArgs(pid)` / `readSeatWindowRect(pid)` reads on-screen layer-0 window bounds via `CoreGraphics.CGWindowListCopyWindowInfo` (`osascript -l JavaScript`) without `System Events` or macOS Accessibility TCC.
- [ ] `placeSeatWindow(pid, rect, options)` places windows via `<instanceHome>/DevToolsActivePort` loopback CDP (`Browser.grantPermissions` `windowManagement` + `Runtime.evaluate` `userGesture: true`) when available before falling back to `System Events`, and reads back the settled frame via `readSeatWindowRect`.
- [ ] `FleetLifecycleService#restoreSeatWindow` forwards `{ instanceHome: record.instanceHome, harnessType: record.harnessType }` to the window-placement function.
- [ ] Unit tests in `test/playwright/unit/ai/services/fleet/seatWindowRect.spec.mjs` and `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` cover the CoreGraphics `readWindowRectArgs`, CDP `placeDevToolsWindowRect` / `placeSeatWindow` primary path and fallback path, and `FleetLifecycleService` option forwarding.

## Verification

- `npx playwright test --config=playwright.config.mjs test/playwright/unit/ai/services/fleet/seatWindowRect.spec.mjs test/playwright/unit/ai/FleetLifecycleService.spec.mjs`


## Timeline

- 2026-10-29T13:54:33Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-29T13:54:34Z @neo-gemini-julia added the `bug` label
- 2026-10-29T13:54:34Z @neo-gemini-julia added the `ai` label
- 2026-10-29T13:54:34Z @neo-gemini-julia added the `architecture` label
- 2026-10-29T14:01:42Z @neo-gemini-julia cross-referenced by PR #1355

