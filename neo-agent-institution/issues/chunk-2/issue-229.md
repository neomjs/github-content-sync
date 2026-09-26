---
id: 229
title: 'The shell denies the engine''s staged popup, so no cockpit pane tears out or pops out'
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - regression
  - security
assignees:
  - neo-opus-grace
createdAt: '2026-09-26T07:25:37Z'
updatedAt: '2026-09-26T07:26:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/229'
author: neo-opus-grace
commentsCount: 0
parentIssue: 7
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
# The shell denies the engine's staged popup, so no cockpit pane tears out or pops out

## Context

@tobiu, 2026-09-26, in the installed Fleet Manager shell: dragging a pane's tab out of the window does not open a window, and neither does the tab header's pop-out action. *"drag seems to be limited to the electron shell."* Both work in `apps/workstation` in Chrome. Lane offered by @neo-fable-clio.

## The Problem

Both gestures go through one host seam, `AgentOS.view.fleet.cockpit.VesselContainer#openTearOutVessel`, which calls `Neo.Main.windowOpen`. At the engine pin (neo `87ac80a6`), `windowOpen` opens a same-origin URL at `about:blank` first. It writes the child's one-time route into the blank realm's `sessionStorage`, then calls `location.replace()` onto `apps/agentos/childapps/widget/index.html?tearout=<item>` (neo `#15573`, 2026-07-19).

The shell's `setWindowOpenHandler` admits only `isHarnessDocumentUrl`, so it denies `about:blank`. From there:
- `window.open` returns `null`.
- `windowOpen` resolves `false`.
- `openTearOutVessel` returns `null`.
- The engine refuses the admission, by design silently, and the gesture falls back in-window.

In Chrome there is no window-open handler, so the same code opens the vessel.

The smoke never saw it. Its popup slice calls `window.open(APP_URL)` with the final URL, which skips the staged path every product popup takes. That slice was proven on 2026-07-10, nine days before the staging existed.

## The Architectural Reality

- `harness/main.mjs#configureWebContents` (dev@4ce50eb): `setWindowOpenHandler` → `isHarnessDocumentUrl(target)` → deny. `will-navigate` uses the same predicate.
- `harness/contentPolicy.mjs#isHarnessDocumentUrl`: `app://neo/apps/agentos/**.html` only.
- `harness/main.mjs#isTrustedIpcSender`: the preload's capabilities answer only a harness-document sender, so an `about:blank` realm gets none.
- ADR 0034 §2.3.2 is the policy these implement. A widened window allowlist amends it first: neomjs/neo#19237.

**Measured on Electron 43.1.0** (the harness pin), with a minimal mirror of `configureWebContents` over the real `contentPolicy.mjs` that replays `windowOpen`'s staging:
- **Today:** `open-handler url=about:blank → deny`, and `window.open → null`.
- **`about:blank` admitted:** `will-navigate` fires for the opener's `location.replace` and admits the widget document. The popup lands there, and its `sessionStorage` carries the staged route.
- **Negative arm:** a staged realm replaced onto `https://example.com/` is prevented and stays blank.

## The Fix

1. `harness/contentPolicy.mjs`: add `isHarnessPopupUrl(value)`, which is `value === 'about:blank' || isHarnessDocumentUrl(value)`. Its JSDoc names the engine staging it serves.
2. `harness/main.mjs#configureWebContents`: `setWindowOpenHandler` admits `isHarnessPopupUrl`. `will-navigate` keeps `isHarnessDocumentUrl`, so the staged realm's only exit is an allowlisted document.
3. The smoke's popup slice opens through `Neo.Main.windowOpen`, the product path, instead of a raw `window.open(APP_URL)`. Every smoke run then exercises the staging it missed.

## Acceptance Criteria

- [ ] `ContentPolicy.spec.mjs` pins `isHarnessPopupUrl`:
  - it admits exactly `about:blank` plus every harness document;
  - it rejects `about:blank#x`, `about:blank?x`, `about:srcdoc`, `data:` / `https:` URLs and non-document `app://` paths;
  - `isHarnessDocumentUrl('about:blank')` stays `false`, so navigation is unchanged.
- [ ] `configureWebContents` uses `isHarnessPopupUrl` for window opens only. `will-navigate` is unchanged.
- [ ] The smoke's popup slice goes through `Neo.Main.windowOpen`, and a local `npm run smoke` at the head boots the popup with the shared-heap evidence intact.
- [ ] Real-Electron receipt in the PR body, red at dev and green at the head, for the staged open. It includes the off-origin negative arm.
- [ ] The ADR 0034 §2.3.2 amendment (neomjs/neo#19237) merges first, or with this PR.

## Post-Merge Validation

- [ ] A team `.app` rebuilt from dev: the cockpit's pop-out header action, and a tab dragged out of the window, each open a vessel holding the live pane.

## Out of Scope

- Engine changes. The staging fixes a real handshake race (neo `#15573`).
- Dock-conversion behaviour over a target popup: neo `#19225` / PR `#19232`, @neo-fable's lane.
- The shell sharing the installed app's `userData` in checkout runs (Ada's defect-note of 2026-09-25).

## Avoided Traps

- **Admitting `about:` broadly, or any URL whose origin reads `null`.** Only the staging realm is needed. Custom-scheme origins read `null` in main, which is how the first shell denied its own popup on 2026-07-10.
- **Loosening `will-navigate` as well.** The staged realm needs admission to be born, not permission to go anywhere.
- **A shell-side re-implementation of the route handshake.** The engine owns it, and the shell admits it.

## Decision Record impact

`depends-on` the ADR 0034 §2.3.2 amendment, neomjs/neo#19237. Implements §2.2 C3 as the engine now performs it.

## Related

- Parent: #7, the Electron shell epic. The symptom sits in the cockpit (#9 / #10); the defect sits in the shell's window policy.
- neo `#15573` introduced the staging.
- neo `#19225` / PR `#19232` are the neighbouring engine conversion defect, a different layer.

Sweeps: live latest-open sweep at 2026-09-26T07:24Z (latest 20 Institution issues, no equivalent; #227 and #228 are unrelated); `gh search issues --owner neomjs "about:blank"` / `"tear-out shell"` found closed engine tickets only; the MC rationale sweep found no prior decision; the A2A in-flight sweep shows @neo-fable-clio's lane offer (07:16Z), accepted by me (07:18Z).

Origin Session ID: 81d1894c-d8fd-4192-8350-42e32eb0101e
Retrieval Hint: "FM shell tear-out popout refused: setWindowOpenHandler denies about:blank staging"


## Timeline

- 2026-09-26T07:25:38Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-26T07:25:39Z @neo-opus-grace added the `bug` label
- 2026-09-26T07:25:39Z @neo-opus-grace added the `agent-os` label
- 2026-09-26T07:25:39Z @neo-opus-grace added the `ai` label
- 2026-09-26T07:25:40Z @neo-opus-grace added the `regression` label
- 2026-09-26T07:25:40Z @neo-opus-grace added the `security` label
- 2026-09-26T07:26:01Z @neo-opus-grace cross-referenced by #19237
- 2026-09-26T07:26:04Z @neo-opus-grace added parent issue #7
- 2026-09-26T07:35:12Z @neo-opus-grace cross-referenced by PR #232
- 2026-09-26T07:35:15Z @neo-opus-grace cross-referenced by PR #19240

