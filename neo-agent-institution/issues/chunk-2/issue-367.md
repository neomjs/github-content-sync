---
id: 367
title: 'The engine pin carries neo#19338, so the installed FM stops drawing while hidden'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - performance
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T19:45:41Z'
updatedAt: '2026-09-30T22:42:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/367'
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
closedAt: '2026-09-30T22:42:35Z'
---
# The engine pin carries neo#19338, so the installed FM stops drawing while hidden

## Context

neomjs/neo#19338 (Resolves neomjs/neo#19336) makes `Neo.app.SharedCanvas` pause its renderer while its window is hidden. That covers every host, and Home and the Observatory are among them. Under `useSharedWorkers` those loops run on `setTimeout(1000 / 60)` today and keep drawing in a minimized Fleet Manager. This repository reaches the change only through its engine pin, `package.json` → `"neo.mjs": "github:neomjs/neo#067f9fb93b28d917c7dfdba645d72b921784c7b3"`, which predates it.

## The Problem

Until the pin moves, the installed Fleet Manager keeps both canvases drawing while hidden, and the verification that neomjs/neo#19336 AC-4 and the removed #366 AC-3 described has no owner that survives their merges.

## The Fix

After neomjs/neo#19338 merges, move the pin to an engine `dev` commit that contains it. Run the unit, e2e and visual tiers at the new pin, then verify the behavior on the installed build.

## Acceptance Criteria

- [ ] AC-1 The engine pin names an engine `dev` commit that contains neomjs/neo#19338's merge, and the unit, e2e and visual tiers pass at it. Bound: `FleetCockpitBarCompositionNL`'s two darwin snapshots fail identically on the old pin, so they are outside this AC. CI skips platform-visual specs, and the red is routed as a defect-note.
- [ ] AC-2 `[L4-deferred — operator handoff needed]` *(post-merge, installed)* With the installed Fleet Manager's window minimized, Home's `getStats().frames` does not advance, and the Observatory's canvas draws no frames. Residual-Owner: neomjs/neo-agent-brain#64 AC-16, which survives this ticket's close; its receipt is appended here too.

## Out of Scope

- Any other engine change the pin move happens to carry. The PR lists them and adapts only what breaks.
- Renderer behavior itself, which neomjs/neo#19338 owns.

## Related

neomjs/neo#19336 · neomjs/neo#19338 · #366 · #360

Decision Record impact: none.
Structure map: N/A. A dependency pin moves; no file or folder is added.
Live latest-open sweep: the latest 20 open issues at 2026-09-30T19:45:13Z, plus an exact `engine pin` search; no open pin ticket.
A2A in-flight claim sweep: no claim on the engine pin today.
Own-assignment sweep: #365, #366 and #244 are mine, and #366's AC-3 moves here.

Blocked by neomjs/neo#19338 (the pin needs its merge commit).

Origin Session ID: eee527e0-79de-4edb-b4c2-eb89d3780d56


## Timeline

- 2026-09-30T19:45:42Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T19:45:43Z @neo-opus-vega added the `enhancement` label
- 2026-09-30T19:45:43Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T19:45:43Z @neo-opus-vega added the `ai` label
- 2026-09-30T19:45:44Z @neo-opus-vega added the `performance` label
- 2026-09-30T19:46:06Z @neo-opus-vega cross-referenced by #366
- 2026-09-30T19:46:08Z @neo-opus-vega cross-referenced by PR #19338
- 2026-09-30T19:47:10Z @neo-opus-vega cross-referenced by PR #368
- 2026-09-30T19:59:36Z @neo-opus-vega cross-referenced by #19336
- 2026-09-30T21:42:28Z @neo-opus-vega cross-referenced by PR #373
- 2026-09-30T22:24:53Z @neo-opus-vega cross-referenced by #64
- 2026-09-30T22:42:35Z @tobiu referenced in commit `a01c6bd` - "Merge pull request #373 from neomjs/vega/367-engine-pin

chore(deps): engine pin → dev@e7d550e5dc, which carries SharedCanvas's pause while its window is hidden (#367)"
- 2026-09-30T22:42:36Z @tobiu closed this issue

