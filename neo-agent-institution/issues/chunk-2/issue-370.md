---
id: 370
title: 'The macOS app icon is the raw logo, so macOS tiles it light and the N crowds the tile'
state: CLOSED
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T21:31:08Z'
updatedAt: '2026-10-01T08:20:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/370'
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
closedAt: '2026-10-01T08:20:39Z'
---
# The macOS app icon is the raw logo, so macOS tiles it light and the N crowds the tile

## Context

@tobiu, 2026-09-30, looking at the Dock: *"should the 'N' => neo logo inside the app icon be a bit smaller?"* He agreed to the recommendation that followed: a dedicated app icon with a dark navy tile and a solid white N at about 60 % of the tile.

**Sweep attestations (2026-09-30T21:30Z):**
- Live latest-open: the latest 20 open issues, no icon ticket. #355 (closed by #356) was the first pass; #12 is the native-shell UX spec.
- A2A: no claim on the app icon.
- MC sweep ("macOS app icon Neo logo harness electron-builder icon dock tile size padding"): the one relevant hit is @neo-gpt's 07-11 note that pins the mark's authority (the exact two-ribbon paths, white fill, `#3E63DD` outline, no redesign).
- Own-assignment: 1 open (#11), not overlapping.

## The Problem

`harness/electron-builder.yml` gives `mac.icon` the brand logo itself, `resources/images/logo/neo_logo_primary.svg` (#356). Its ribbons span x 8.8–144.9 and y 8.9–149.3 of a 157 canvas, so the logo is full bleed. It is no app icon:
- **The Dock wraps it in the system's own light tile.** Rendered through `NSWorkspace.icon(forFile:)` on macOS 27.0.1, a bundle carrying this icon shows the logo inside a light-grey squircle the system supplies.
- **The N crowds that tile.** It spans about 80 % of the tile's width. In the operator's Dock, the ChatGPT and Claude marks sit at about 60–65 %.
- **The white fill disappears on the light tile**, so the mark reads as a thin blue outline.

## The Fix

A dedicated icon source, `harness/assets/app/neoAppIcon.svg`, beside the tray art in `harness/assets/tray/`. It follows Apple's macOS grid:
- a 1024 canvas with an 824 rounded tile, radius 185.4, and a soft drop shadow;
- a navy tile taken from the brand blue `#3E63DD` (same hue, `#1c2c5f` → `#0d1535`, lit from the top);
- the mark's exact two paths, filled white, at about 60 % of the tile's width and centred on the mark's own point of symmetry.

The mark is not redesigned. The white single-colour rendering of the same paths is the one that stays legible on a dark tile. With the outline kept, the blue edges swamp the white at 32 px and 16 px.

`mac.icon` points at the new source. The in-app logo (`contentPolicy.mjs`, `prepareAssets.mjs`) keeps `neo_logo_primary.svg`.

## Acceptance Criteria

- [ ] AC-1: `mac.icon` names `assets/app/neoAppIcon.svg`, which holds the mark's paths unchanged.
- [ ] AC-2: Rendered by the system (`NSWorkspace.icon(forFile:)` on a bundle carrying the built `.icns`), the icon shows its own navy tile with no system-supplied light tile around it.
- [ ] AC-3: The N spans 58–62 % of the tile's width and stays legible at 32 px and 16 px.
- [ ] AC-4 *(post-merge)*: the operator's rebuilt app shows the new icon in the Dock.

## Out of Scope

- The in-app logo, the tray art, and the Windows and Linux targets, which have no icon configured.

## Related

- #355 / #356 (the first pass), #12 (the native-shell UX spec), #7 (the Electron shell)

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636

## Timeline

- 2026-09-30T21:31:08Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T21:31:10Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T21:31:10Z @neo-opus-grace added the `ai` label
- 2026-09-30T21:31:10Z @neo-opus-grace added the `design` label
- 2026-09-30T21:34:46Z @neo-opus-grace cross-referenced by PR #371
- 2026-09-30T22:00:21Z @neo-fable-clio cross-referenced by #374
- 2026-09-30T22:24:06Z @neo-opus-grace cross-referenced by #375
- 2026-10-01T08:20:39Z @tobiu referenced in commit `5a81674` - "feat(harness): the macOS app icon is its own navy tile with the Neo mark in white at 60 % (#370) (#371)

mac.icon named the full-bleed brand logo, which macOS wraps in a light system tile where the N fills about 80 % and its white fill disappears. assets/app/neoAppIcon.svg follows Apple's 1024/824 grid: a navy tile from the brand blue #3E63DD, the mark's exact paths in white at 60 % of the tile width. Verified through electron-builder 26.15.3's own icons toolset (icons@1.1.0, resvg) and NSWorkspace on macOS 27.0.1: the system renders the new tile as-is."
- 2026-10-01T08:20:39Z @tobiu closed this issue

