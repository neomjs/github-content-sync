---
id: 386
title: The cockpit window draws a gray native title bar above its own dark top bar
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T13:45:28Z'
updatedAt: '2026-10-02T08:31:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/386'
author: neo-opus-grace
commentsCount: 0
parentIssue: 12
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-02T08:31:06Z'
---
# The cockpit window draws a gray native title bar above its own dark top bar

## Context

The operator, 2026-10-01, comparing the installed Fleet Manager with Claude and Codex: the default gray Electron title bar does not fit. Both of those apps draw their own header and place the traffic lights inside it. This revises #12's v1 line, "No custom chrome in v1 — the web surface is the product; the shell frames it honestly", for macOS. The revision keeps #12's principle: in a browser the cockpit is unchanged, and only the shell's frame moves into the cockpit's own top bar.

## The Problem

`harness/main.mjs` `createHarnessWindow` opens the main window with defaults (`width`, `height`, secure `webPreferences`). macOS therefore draws its native title bar, a gray strip titled "Neo.mjs Agent Institution", above the cockpit's dark `agent-top-toolbar`, which already carries the logo, "Agent OS", the instance switcher and the theme button.

## The Architectural Reality

- macOS offers no colour for the native bar, only light/dark via `nativeTheme` or hiding it. With `titleBarStyle: 'hiddenInset'` plus `titleBarOverlay: true`, Electron 43 keeps the traffic lights over the web content and publishes the window-controls-overlay geometry as CSS `env(titlebar-area-*)`.
- Measured with the harness's own Electron 43.5.0 (throwaway probe, lights at `{x: 16, y: 17}`): `titlebar-area-x` = 92px, height 48px, `navigator.windowControlsOverlay.visible` = true. A bar padded by `max(9px, env(titlebar-area-x, 9px))` starts its content right after the lights; a browser reports no overlay, so the fallback keeps today's 9px.
- `resources/scss/src/apps/agentos/Viewport.scss` `.agent-top-toolbar`: `min-height: 50px`, left padding a recorded 9px exception for the mark. Its interactive children are `.neo-button`s: the theme button, and the instance switcher (`SwitcherButton`, `fm-instance-switcher`). The flex spacer between them stays draggable.

## The Fix

1. `createHarnessWindow` on macOS: `titleBarStyle: 'hiddenInset'`, `titleBarOverlay: true`, `trafficLightPosition` centred in the 50px bar.
2. `.agent-top-toolbar`: left padding `max(9px, env(titlebar-area-x, 9px))`, `-webkit-app-region: drag`, and `no-drag` on its `.neo-button`s.
3. Windows and Linux keep the native frame: their overlay draws window controls in fixed colours, which a theme switch would need to recolour (a later leaf). Popped-out vessel windows keep theirs too; they have no top bar to host the lights.

## Acceptance Criteria

- [x] AC-1: on macOS the main window shows no native title bar; the traffic lights sit vertically centred in the cockpit's top bar, with the logo after them (screenshot of the running shell).
- [x] AC-2: the bar drags the window, and every control in it still clicks: the instance switcher, the theme button.
- [x] AC-3: in a browser the cockpit renders as before: the visual baselines pass unchanged.
- [ ] AC-4 (installed): the packaged app shows the same frame. Residual owner: #7.

AC-1 to AC-3 were delivered by PR #387 at `cd21edf`, merged as `1aadacd` on 2026-10-02 after cross-family approval by @neo-gpt-sophie. The evidence rows and the probe runs' build mapping are in the PR body. AC-4 needs the packaged app and stays with #7.

## Out of Scope

Windows/Linux overlay colours; popped-out windows; custom window controls.

## Decision Record impact

`none` as an ADR. It revises #12's "no custom chrome in v1" for macOS on the operator's request.

## Related

#12 (native shell UX spec), #13 (design conformance), #9.

## Sweeps

Live latest-open sweep: latest 20 open Institution issues at 2026-10-01T13:44:47Z, no equivalent. `gh search issues` "title bar", "titlebar", "traffic lights", "window chrome": #12's v1 decision (above) and closed credential-window items, nothing open on this. A2A in-flight sweep (latest 13:34Z): no claim. Own-assignment sweep: #380 and #11, neither overlapping.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364
Retrieval Hint: "Electron hiddenInset titleBarOverlay env(titlebar-area-x) cockpit top bar traffic lights macOS"

🖖 Grace · @neo-opus-grace · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-01T13:45:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T13:45:30Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T13:45:30Z @neo-opus-grace added the `ai` label
- 2026-10-01T13:45:31Z @neo-opus-grace added the `design` label
- 2026-10-01T13:45:35Z @neo-opus-grace added parent issue #12
- 2026-10-01T14:11:45Z @neo-opus-grace cross-referenced by PR #387
- 2026-10-01T14:14:21Z @neo-opus-grace cross-referenced by #388
- 2026-10-01T14:18:15Z @neo-fable-clio cross-referenced by #389
- 2026-10-01T14:22:53Z @neo-fable cross-referenced by #392
- 2026-10-01T16:28:58Z @neo-opus-grace cross-referenced by #396
- 2026-10-02T08:29:50Z @neo-opus-grace cross-referenced by #414
- 2026-10-02T08:30:43Z @neo-opus-grace cross-referenced by #415
- 2026-10-02T08:31:06Z @tobiu referenced in commit `1aadacd` - "feat(harness): the cockpit's top bar is the macOS window's title bar (#386) (#387)

On macOS the main window hides its native bar (hiddenInset + titleBarOverlay): the traffic lights sit centred in the 50px top bar, which pads its content past them with env(titlebar-area-x) and drags the window; its buttons stay no-drag. A browser reports no overlay, so the 9px inset and every golden stay as they were; the visual stamp follows the SCSS blob."
- 2026-10-02T08:31:06Z @tobiu closed this issue

