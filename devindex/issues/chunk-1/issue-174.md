---
id: 174
title: Measure opt-in gesture recognition on the 50k component-cell grid
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-11-27T21:23:43Z'
updatedAt: '2026-11-27T22:30:25Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/174'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 22400
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
# Measure opt-in gesture recognition on the 50k component-cell grid

## Context

DevIndex is the v14.1 gesture workstream’s real large-grid acceptance case. Its existing [device benchmark](https://github.com/neomjs/devindex/blob/dev/apps/devindex/resources/device-benchmark.mjs), [Grid scroll benchmark](https://github.com/neomjs/devindex/blob/dev/test/playwright/e2e/benchmarks/GridScrollBenchmark.spec.mjs) and 50k streaming Contributors store already exercise buffered rows and component-based cells. The earlier VDom A/B data measured worker placement. It cannot answer how much a gesture subscriber adds to a scroll. [Engine input decision](https://github.com/neomjs/neo/issues/22400)

## The Fix

Use the same dataset, grid dimensions and existing benchmark journeys to compare recognition with no gesture subscriber and with an opted-in surface. Isolate raw-only, semantic-only, both and neither for Main-to-App counts. Preserve the intentional held-drag fling. Keep browser policy, VDom setting, device power state and travel fixed while comparing recognition. Reuse existing HUD/reporting rather than adding a second benchmark app.

## Acceptance Criteria

- [ ] AC-1 Record candidate SHA, device, engine/browser version, power state, viewport, visible row/column buffers, actual input provider, active contacts, realized travel and any discarded/invalid run. A zero-subscriber run adds zero gesture recognition messages above the raw forwarding baseline.
- [ ] AC-2 Report raw and semantic messages, frame-time tails and visible blank time with actual qualifying sample counts and uncertainty on the component-cell grid. Quick native scroll, held JS drag and its release inertia must keep one scroll owner. Report regressions and inconclusive comparisons explicitly.
- [ ] AC-3 Run the supported browser control and preserve a physical iOS/Android comparison for policy decisions. Check page scroll/zoom outside the grid and a second finger arriving mid-scroll. Link the evidence to the Engine v14.1 parent and to Ada’s physical policy arm.
 
The baseline can run on the existing DevIndex build. The candidate subscriber comparison needs a build pinned to the Engine’s per-window registration (neomjs/neo#22403) and a working pan recognizer (neomjs/neo#22405). Record that Engine commit with the counts. A baseline-only run leaves the candidate comparison open.

Related: neomjs/neo#22400, neomjs/neo#22402, #169. Production recognizers and browser-default changes are Engine work.
 
Origin Session ID: c3328164-c307-4c22-b385-606a978d46f0

Retrieval Hint: DevIndex 50k gesture recognition subscriber counts blank time physical mobile

## Timeline

- 2026-11-27T21:23:44Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-27T21:23:44Z @neo-gpt-emmy added the `ai` label
- 2026-11-27T21:24:06Z @neo-gpt-emmy added parent issue #22400
- 2026-11-27T21:44:16Z @neo-gpt-euclid cross-referenced by #22400

