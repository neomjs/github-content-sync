---
id: 355
title: Use the Neo logo as the packaged macOS app icon
state: CLOSED
labels:
  - bug
  - ai
  - build
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T14:21:28Z'
updatedAt: '2026-09-30T15:44:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/355'
author: neo-gpt-emmy
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
closedAt: '2026-09-30T15:44:28Z'
---
# Use the Neo logo as the packaged macOS app icon

## Context

During installed FM onboarding on 2026-09-30, the operator showed macOS Files & Folders rendering Neo Harness with Electron's default icon, and asked for the Neo logo already used in the system/header toolbar.

## The Problem

The shell has a branded in-app header and tray assets, but its macOS application bundle still uses the packager's fallback. The current build explicitly reports `default Electron icon is used` because the application icon is unset.

## The Architectural Reality

`apps/agentos/view/Viewport.mjs:133` references the existing canonical `resources/images/logo/neo_logo_primary.svg`. `harness/electron-builder.yml` owns bundle packaging and currently declares no application icon. The installed electron-builder 26.15.3 supports SVG-to-ICNS conversion; the [v26 icon documentation](https://www.electron.build/v26/docs/features/icons-and-images/) confirms SVG input. `harness/assets/tray` is a separate menu-bar surface and is not the app bundle icon.

## The Fix

Set the macOS icon path in the existing packaging configuration to the existing Neo SVG. Let electron-builder perform its built-in conversion; do not copy/redraw the logo or add a separate generation pipeline. Build a candidate without replacing the live app that owns the new peer's process.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Packaged macOS app icon | Existing Neo header SVG; operator request | Bundle icon derives from that SVG | A missing/invalid configured source must fail packaging rather than silently ship Electron branding | Packaging config points to source | Candidate Info.plist, bundled ICNS and rendered icon |

## Acceptance Criteria

- [ ] AC-1: The packaging configuration uses the existing header-logo SVG as the single artwork source.
- [ ] AC-2: A packaged candidate contains the derived icon and its Info.plist references it; the build no longer reports the default Electron icon fallback.
- [ ] AC-3: Inspect the candidate's rendered icon to confirm the Neo mark. Installed Dock/Settings cache refresh is post-merge validation under #12 and must not interrupt an active peer solely for cosmetic acceptance.

## Out of Scope

Tray icon changes, new artwork, TCC permission behavior (#354), code signing, app renaming, and changing another maintainer's running profile.

## Related

#12 · #7 · #354

Latest 20 open issues, exact icon/logo history and recent all-state A2A show no equivalent. Own-assignment sweep: no open assignments here before this claim. Memory recall on default Electron/app icon nouns returned unrelated initialization material. Existing packaging file owns the setting; no new source module. Decision Record impact: none.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e
Retrieval Hint: packaged Neo Harness macOS application icon Electron default header SVG.

## Timeline

- 2026-09-30T14:21:29Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T14:21:30Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T14:21:30Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T14:21:30Z @neo-gpt-emmy added the `build` label
- 2026-09-30T14:24:42Z @neo-gpt-emmy cross-referenced by PR #356
- 2026-09-30T14:57:16Z @tobiu referenced in commit `3825816` - "Merge pull request #356 from neomjs/codex/355-native-app-icon

fix(harness): use the Neo logo for the macOS app (#355)"
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:44:28Z @tobiu closed this issue

