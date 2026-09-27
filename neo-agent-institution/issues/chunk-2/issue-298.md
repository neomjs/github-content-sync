---
id: 298
title: The theme switch does nothing on its first click
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T14:25:26Z'
updatedAt: '2026-09-27T15:39:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/298'
author: neo-opus-grace
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
closedAt: '2026-09-27T15:39:58Z'
---
# The theme switch does nothing on its first click

## Context
On a fresh profile whose OS does not prefer dark, the FM's theme switch does nothing on its first click; the second click switches. Seen in a real browser (headless Chromium, the #291 branch): the viewport carried `neo-theme-neo-light` only after the second click. The visual spec already works around it: `FleetCockpitVisual.spec.mjs:299` and `:462` click up to three times, "the first click only makes the config default explicit".

For a first-run user this is the first control they try in the top bar, and it looks broken.

## The Problem
`ViewportController.onComponentConstructed` applies a theme only when one is stored (`agentosTheme`) or `Neo.config.prefersDarkTheme` is set. Otherwise `viewport.theme` stays unset, and the app renders its first configured theme, `neo-theme-neo-dark` (`apps/agentos/neo-config.json` `themes`). `onSwitchTheme` reads an unset theme as `neo-theme-neo-light`, so the first click sets dark over dark.

Both branches date from the initial files. No decision on the default is recorded, and the FM ships dark-first: dark is `themes[0]`, the goldens are dark, and a dark-preferring OS gets dark explicitly.

## The Architectural Reality
- `apps/agentos/view/ViewportController.mjs` `onSwitchTheme`: `oldTheme = viewport.theme || 'neo-theme-neo-light'`.
- `onComponentConstructed`: stored value → `setTheme(value, false)`; else `prefersDarkTheme` → `setTheme('neo-theme-neo-dark', false)`; else nothing.
- The engine already resolves the theme on screen: `Neo.component.Base#getTheme` answers the component's own theme, then the nearest ancestor's, then the window's or app's `themes[0]`. `Neo.component.Canvas#resolveColorScheme` documents this bug class: reading the `theme` config "answers `null` on first render and the right value only after a theme toggle".

## The Fix
`onSwitchTheme` reads the current theme through `viewport.getTheme()` instead of the `theme` config with a light fallback. The boot and the stored-theme path stay unchanged, so nothing renders differently.

## Acceptance Criteria
- [ ] AC-1: On a fresh profile without an OS dark preference, one click on the theme switch puts `neo-theme-neo-light` on the viewport. Real browser: `FleetCockpitVisual`'s light arms click once and assert it, red on dev.
- [ ] AC-2: A stored theme and a dark-preferring OS still apply at boot, and further clicks alternate dark and light.

## Out of Scope
- Honouring a light-preferring OS with the light skin at boot. That changes the product's default look and is a design call.
- The switch's icon and tooltip.

## Related
- #10 (FM cockpit UI/UX), polish item of the operator's four release goals.
- Found while verifying #291's light skin.

Live latest-open sweep: checked the latest 20 open issues at 14:24Z; no equivalent found (#267 names the theme switch only as a tooltip precedent). A2A claim sweep: no claim on this scope in the last hour. Memory Core sweep ("theme switch first click", `prefersDarkTheme`): no prior decision; `git log -S` puts both branches in the initial files. Own-assignment sweep: no same-surface ticket.

Origin Session ID: 0dc6daad-2744-44c9-91cb-38d82e9e82e6


## Timeline

- 2026-09-27T14:25:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T14:25:27Z @neo-opus-grace added the `bug` label
- 2026-09-27T14:25:27Z @neo-opus-grace added the `ai` label
- 2026-09-27T14:47:26Z @neo-opus-grace cross-referenced by PR #299
- 2026-09-27T15:24:12Z @tobiu referenced in commit `4a4a383` - "fix(agentos): the theme switch reads the theme on screen, so its first click switches (#298)"
- 2026-09-27T15:39:58Z @tobiu referenced in commit `37e2720` - "Merge pull request #299 from neomjs/grace/298-theme-first-click

fix(agentos): the theme switch reads the theme on screen, so its first click switches (#298)"
- 2026-09-27T15:39:58Z @tobiu closed this issue

