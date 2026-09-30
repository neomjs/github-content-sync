---
id: 365
title: Home's field goes quiet under the hero's lines
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T15:28:11Z'
updatedAt: '2026-09-30T15:28:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/365'
author: neo-opus-vega
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
# Home's field goes quiet under the hero's lines

## Context

Grace's AC-4 design review of #360 (2026-09-30) measured the limit of the text-shadow halo behind Home's hero lines. It lifts the ground at the glyph strokes by up to 13 levels. But a mote in the band just above the x-height, the one over the "i" of "streaming" in the first-run light golden, is pixel-identical with and without the halo. At 2x, motes and links sit in the lines' leading like diacritics, and in motion they drift through the words. No glyph-shaped shadow can reach that space. #244 recorded it as a known v1 limit, which the portal hero shares.

## The Problem

Two layers that know nothing of each other paint Home. The canvas worker draws the field, and the DOM draws the hero's lines over it. Only the host knows where the lines are, so the renderer cannot keep the field off them.

## The Architectural Reality

- `AgentOS.canvas.Home` (`apps/agentos/canvas/Home.mjs`) draws, each frame, the haze, links and motes, then the team's marks, then the ripples.
- `AgentOS.view.home.Canvas` (`apps/agentos/view/home/Canvas.mjs`) forwards the team, stillness, size and pointer. It measures its own viewport rect on every size report.
- The hero's lines are `.fm-home-eyebrow`, `.fm-home-h1`, `.fm-home-lede` and `.fm-home-plane` in `AgentOS.view.home.Container`. Their boxes hug the text (vbox, `align: 'start'`), and `applyState` changes them (the reader switch, the team line's text).

## The Fix

The host forwards the lines' rects, canvas-relative in CSS pixels, as a `quiet` input. It measures them when the canvas becomes ready, on every resize, and after `applyState` changes the lines. The renderer erases the field under those rects with a feathered edge (`destination-out`) after links, motes and ripples. It then draws the haze underneath (`destination-over`) and the marks on top, so no mote or link sits on a glyph and no mark is dimmed.

## Acceptance Criteria

- [ ] AC-1 Inside a line's rect plus its feather, no mote or link is drawn, while marks and the haze are unchanged (renderer unit arm on the erase pass; `getStats` counts the quiet rects).
- [ ] AC-2 The rects follow the lines: a reader switch (first run ↔ returning) and a resize both move them (e2e arm through the field driver).
- [ ] AC-3 A 2x crop of the first-run lede and the plane line shows no mote on or inside a glyph's ascender band in either skin; goldens re-recorded and reviewed by the design owner.

## Out of Scope

- Thinning the field near the lines instead of erasing it under them.
- Removing the halo: it stays, since it lifts the ground at the strokes.

## Avoided Traps

- **A box-shaped CSS clearing:** it drew a hard edge at its box and dimmed two lit marks (the first AC-4 review of #244).
- **More text-shadow layers:** measured at six layers; they cannot reach the leading.

## Related

#244 · #360

Decision Record impact: none.
Structure map: N/A. The change is the Institution app, which the Brain's structure map does not host.
Live latest-open sweep: the latest 20 open issues at 2026-09-30T15:27:42Z; no equivalent found.
A2A in-flight claim sweep (all read states, last 15): no claim on Home's field or its hero lines.
MC sweep: "motes sit on the lede like diacritics Home field halo text-shadow cannot reach leading", 5 results, no prior decision found.
Own-assignment sweep: 2 open (#358, #244); #244's Out of Scope holds only the pointer this ticket replaces.

Origin Session ID: 558684c5-baee-46e8-baea-71bdf90dbce1
Retrieval Hint: `query_raw_memories("Home field quiet rects hero lines motes leading halo canvas worker")`


## Timeline

- 2026-09-30T15:28:12Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T15:28:12Z @neo-opus-vega added the `enhancement` label
- 2026-09-30T15:28:13Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T15:28:13Z @neo-opus-vega added the `ai` label
- 2026-09-30T15:28:13Z @neo-opus-vega added the `design` label
- 2026-09-30T15:28:31Z @neo-opus-vega cross-referenced by #244
- 2026-09-30T15:28:33Z @neo-opus-vega cross-referenced by PR #360
- 2026-09-30T18:26:05Z @neo-opus-vega cross-referenced by #366

