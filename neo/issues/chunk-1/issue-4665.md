---
id: 4665
title: 'replace all occurrences of setTimeout() with core.Base: timeout()'
state: OPEN
labels:
  - enhancement
assignees: []
createdAt: '2023-08-06T17:50:03Z'
updatedAt: '2026-11-05T10:30:51Z'
githubUrl: 'https://github.com/neomjs/neo/issues/4665'
author: tobiu
commentsCount: 4
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
# replace all occurrences of setTimeout() with core.Base: timeout()

this one might take a while :)

## Timeline

- 2023-08-06T17:50:03Z @tobiu added the `enhancement` label
### @github-actions - 2024-08-29T02:26:50Z

This issue is stale because it has been open for 90 days with no activity.

- 2024-08-29T02:26:51Z @github-actions added the `stale` label
### @github-actions - 2024-09-13T02:29:20Z

This issue was closed because it has been inactive for 14 days since being marked as stale.

- 2024-09-13T02:29:21Z @github-actions closed this issue
### @neo-opus-grace - 2026-10-09T20:38:35Z

#19489 set C · T6 · Grace · 2026-10-09: **reopened, narrowed.** About 60 `setTimeout(` calls remain in `src`; classes with a `core.Base` can switch to `me.timeout()` in slices.

- 2026-10-09T20:41:46Z @neo-opus-grace cross-referenced by #19489
- 2026-11-05T10:20:03Z @neo-opus-grace cross-referenced by #15000
### @neo-gemini-julia - 2026-11-05T10:30:51Z

### Backlog Premise Read (`2026-11-05` · `neomjs/neo#15000`)

- **Verdict:** **rewrite (narrowed scope · operator-authored)**
- **Premise check against `dev`:**
  - `core.Base#timeout(time, {signal}={})` (`src/core/Base.mjs:1205-1231`) returns a **`Promise`** registered via `me.registerAsync(id, fail)` (which rejects with `Neo.isDestroyed` if the instance is destroyed while pending, or `signal.reason` when aborted), rather than returning a numeric timer ID for `clearTimeout(id)`.
  - Auditing all 49 `setTimeout(` call sites across `src/` breaks them into three distinct groups:
    1. **Out of scope (do not extend `core.Base` — 16 sites):** `src/Main.mjs` (`lines 450, 671, 688, 922, 994, 1039, 1158, 1233, 1392`), `src/util/Function.mjs` (`lines 34, 115, 119, 122, 197`), `src/util/Rectangle.mjs:631`, and `src/main/DomEvents.mjs:609`.
    2. **Direct 1:1 `this.timeout()` replacements in `core.Base` subclasses (3 sites):**
       - `src/mixin/VdomLifecycle.mjs:274` (`await new Promise(resolve => setTimeout(resolve, 1))` → `await this.timeout(1)`)
       - `src/worker/App.mjs:670` (`this.isSharedWorker && setTimeout(() => this.reapWindow(data.windowId))`)
       - `src/canvas/Base.mjs:358` (`setTimeout(me.waitForCanvas.bind(me, canvasId, windowId, hasChange), 50)`)
    3. **Re-entrant cancellable timers storing numeric `timeoutId` for `clearTimeout(id)` (29 sites):** `src/app/header/Canvas.mjs:175`, `src/button/Base.mjs:608`, `src/button/Effect.mjs:423`, `src/calendar/view/calendars/EditContainer.mjs:181`, `src/calendar/view/month/Component.mjs:713`, `src/canvas/GraphScene.mjs:853`, `src/canvas/Header.mjs:286`, `src/canvas/Sparkline.mjs:304`, `src/component/Gallery.mjs:685`, `src/component/Helix.mjs:490,1137`, `src/dashboard/dock/interaction/RevealOverlay.mjs:304`, `src/dashboard/dock/persistence/TopologyLibrary.mjs:233`, `src/grid/plugin/AnimateRows.mjs:362`, `src/list/plugin/Animate.mjs:907`, `src/main/addon/DockFlip.mjs:574,893`, `src/main/addon/GridDragScroll.mjs:292`, `src/main/addon/ResizeObserver.mjs:193`, `src/main/addon/ScrollSync.mjs:94`, `src/main/draggable/sensor/Mouse.mjs:178`, `src/main/draggable/sensor/Touch.mjs:206`, `src/manager/DomEvent.mjs:640`, `src/manager/DragCoordinator.mjs:353,464,600`, `src/manager/Transaction.mjs:966`, `src/manager/VDomUpdate.mjs:574`, `src/menu/List.mjs:372`, `src/tooltip/Base.mjs:92`, `src/worker/CanvasGroups.mjs:181`. Converting these to `this.timeout(ms, {signal})` would require allocating an `AbortController` per timer and `.catch()` handling for abort/destroy rejections.
- **Recommendation for `@tobiu`:** Either narrow this ticket to group (2) (`src/mixin/VdomLifecycle.mjs:274`, `src/worker/App.mjs:670`, `src/canvas/Base.mjs:358`) or close if raw `setTimeout`/`clearTimeout` remains the intended pattern for high-frequency cancellable timers.

🏛️ **Julia** (`@neo-gemini-julia` · Gemini 4 Argon · Antigravity)


