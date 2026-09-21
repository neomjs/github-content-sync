---
id: 92
title: 'Dock headers paint the theme''s green-grey band on the cockpit''s blue-black panel: declare the cockpit''s own inline-header ground'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T09:57:09Z'
updatedAt: '2026-09-04T12:57:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/92'
author: neo-fable-clio
commentsCount: 0
parentIssue: 13
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-04T12:57:15Z'
---
# Dock headers paint the theme's green-grey band on the cockpit's blue-black panel: declare the cockpit's own inline-header ground

## Context

Measured on the served cockpit at engine pin `35d468b51f` (PR #91, 2026-09-04), neo-dark: every dock tab header (`.neo-tab-container-inline > .neo-tab-header-toolbar` — FLEET, ACTIVITY/TASKS/…, AGENT DETAIL, PERSPECTIVES) computes `background-color: rgb(41,45,40)` with an inset hairline `rgb(69,75,66)`. Those are the neo-dark theme's `--tab-header-inline-background-color: var(--sem-color-surface-neutral-highlighted)` and `--tab-header-inline-box-shadow: inset 0 -1px 0 var(--sem-color-border-default)` (engine `resources/scss/theme-neo-dark/tab/Container.scss:13-15`, introduced with the flat inline header, neomjs/neo#18095). The cockpit's own palette around them is blue-black: `--fm-panel rgb(20,26,35)`, `--fm-rail rgb(14,19,26)`, `--fm-line rgb(38,47,61)`, viewport `rgb(11,14,19)`. A green-grey band (hue ≈ 100°) sits on a blue (hue ≈ 215°) surface on every pane.

The stored cockpit goldens carried the same hue since before pin 4 (the pre-flat gradient ran `rgb(41,45,40)` → `rgb(15,17,14)`), so the design gate never saw a change — the hue was inherited from the theme from the start, never chosen. neo-light paints the header white on `rgb(242,245,249)`, which reads fine; the defect is the dark skin's.

## The Problem

`resources/scss/src/apps/agentos/Viewport.scss:306-317` (the `.neo-tab-header-toolbar.neo-dock-top` block) states the cockpit's intent — "the cockpit's strips are flat panel surfaces … opt out at the token" — but opts out of the background **image** only (`--tab-header-inline-background-image: none`). The background **colour** and the hairline were never declared, because when the block was written the engine's colour token fell back to `transparent`. Since neomjs/neo#18141 ("a fallback is not a reset"), every theme states the inline header's paint explicitly, so an undeclared consumer inherits the theme's surface — exactly what the cockpit shows.

## The Architectural Reality

- The engine contract (`resources/scss/src/tab/Container.scss:16-19`): `background-color: var(--tab-header-inline-background-color, transparent)`, `background-image`, `box-shadow: var(--tab-header-inline-box-shadow, none)` — three tokens a consumer sets on the toolbar's nearest ancestor. The cockpit already rebinds `--tab-button-*` on the same block, and after neomjs/neo#18145 the consumer block outranks the theme's value file (that is how the cockpit's 30px header started winning at pin 5).
- The design SSOT (`apps/agentos/VisualSystem.md`, `TOKENS.md`): panel surfaces on `--fm-panel`, edges on `--fm-line`; the strip is chrome, never a highlighted surface.
- Goldens: the six cockpit goldens + the twelve synthesis goldens are stamp inputs; a paint change re-captures them (Darwin) and restamps.

## The Fix

In the `.neo-tab-header-toolbar.neo-dock-top` block: declare `--tab-header-inline-background-color: var(--fm-panel)` (or `transparent` if the design read prefers the pane ground showing through — decide on the screenshot, both themes) and `--tab-header-inline-box-shadow: inset 0 -1px 0 var(--fm-line)`; update the block's comment to name all three tokens as the opt-out. Re-capture the goldens, read the diff by eye, restamp.

## Acceptance Criteria

- [ ] AC-1 On the served cockpit, neo-dark, every dock tab header computes a background colour from the cockpit's palette (`--fm-panel` or the pane ground) and a hairline of `--fm-line`; no `rgb(41,45,40)` / `rgb(69,75,66)` anywhere on the surface. neo-light reads unchanged or better (state the before/after values in the PR).
- [ ] AC-2 The cockpit's header geometry is untouched (30px, the 2px signal underline); `test-visual` 6/6 and the synthesis capture green after re-capture; `check-visual-baselines` green on the pushed tree.
- [ ] AC-3 Before/after evidence in the PR body against the SSOT frame (the #13 leaf convention): the dark pair as the committed `cockpit-default-shell.png` goldens at the before/after heads (embedded by blob URL), light as measured header/hairline values — the visual suite captures dark only, and light was already on the cockpit plate.

## Out of Scope

- The engine's theme tokens themselves (the neo themes' choice of `surface-neutral-highlighted` for a generic inline header is the theme's, not the cockpit's).
- The focus-gated action set on the headers (decided on PR #91: the engine's default set stays).

## Related

#13 (parent, the conformance epic) · #90 / PR #91 (where the measurement was taken) · #81 / PR #82 (the flat header pin) · neomjs/neo#18095 · neomjs/neo#18141 · neomjs/neo#18145

Live latest-open sweep: checked the latest 20 open institution issues at 2026-09-04T09:55Z (#90 … #11); no header-paint ticket; #13's body lists the viewport chrome leaf (the app header + settings affordance), not the dock strips. A2A: no claim on this surface in the last hour (the only institution lane in flight is mine, #90). Memory Core: no prior decision on the dock header ground surfaced (the pin-4 read measured `background-image` only). Structure-map gate: N/A. Structural pre-flight: N/A (SCSS only).

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: "cockpit dock header ground neutral-highlighted green-grey fm-panel tab-header-inline-background-color"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-09-04T09:57:10Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T09:57:12Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T09:57:12Z @neo-fable-clio added the `agent-os` label
- 2026-09-04T09:57:12Z @neo-fable-clio added the `ai` label
- 2026-09-04T09:57:12Z @neo-fable-clio added the `design` label
- 2026-09-04T10:28:27Z @neo-fable-clio cross-referenced by PR #94
- 2026-09-04T12:14:53Z @neo-fable-clio referenced in commit `e594eff` - "fix(agentos): the dock headers paint the cockpit's own plate and edge, never the theme's highlighted band (#92)"
- 2026-09-04T12:14:53Z @neo-fable-clio referenced in commit `5fbdc46` - "test(agentos): the four cockpit goldens re-captured on top of the two-line fleet head; restamp (#92)"
- 2026-09-04T12:32:58Z @neo-fable-clio referenced in commit `9752a8d` - "test(agentos): fleet-grid-cards keeps the merged golden — two pixels of capture noise are not a baseline (#92)"
- 2026-09-04T12:57:15Z @tobiu referenced in commit `ad0de0d` - "Merge pull request #94 from neomjs/agent/92-dock-header-ground

fix(agentos): the dock headers paint the cockpit's own plate and edge, never the theme's highlighted band (#92)"
- 2026-09-04T12:57:15Z @tobiu closed this issue

