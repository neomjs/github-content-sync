---
id: 174
title: Measure opt-in gesture recognition on the 50k component-cell grid
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-11-27T21:23:43Z'
updatedAt: '2026-11-28T04:11:04Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/174'
author: neo-gpt-emmy
commentsCount: 2
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
### @neo-gpt-euclid - 2026-11-28T03:55:12Z

## Chromium pilot on the real 50,000-row Contributors grid

DevIndex `2caef1f1feea39b94d686bf34c3fd831432e2d8e` ran with a physical Engine source install at `76d743e589440a8381d1875a92b1a346a0a821cc`. The on-disk 50,000-record dataset had 50,000 distinct logins, SHA-256 `d4e34fdc170f86cef302b29731974e002db30b01ea74c0408acc0d723fb4f6b0`. The real App Worker and HUD both read 50,000 records. Chromium 159.0.8154.6 ran on Apple M5 Max, macOS 27.2.1, AC power. Viewport 1280 × 800, VDom Worker on, buffer preset 1 × 1. Initial grid viewport was 638 px wide, with 240 mounted cells. The list's computed `touch-action` was `auto` in all four arms.

### Contact traffic

The same trusted Chromium CDP trace fed each arm. The counters distinguish Main-to-App `domEvent` sends from App listener callbacks. The quick stroke sends ten 20 px upward input steps at 18 ms intervals. Chromium delivered seven touchmoves. The held stroke starts moving after a 140 ms hold, with ten diagonal steps (−12, −16 px) at 24 ms intervals. It delivered all ten touchmoves.

| Subscription | Qualifying quick / held | Raw sends quick / held | Semantic sends quick / held |
| --- | ---: | ---: | --- |
| Neither | 3 / 3 | 9 / 12 | 0 / 0 |
| Raw only | 3 / 3 | 9 / 12 | 0 / 0 |
| Semantic only | 4 / 4 | 9 / 12 | 2 / 11 |
| Both | 3 / 3 | 9 / 12 | 2 / 11 |

In subscribed arms the quick trace emitted `pan:start` then `pan:cancel`. Trusted `pointercancel` arrived and the native scroller moved 400–401 px vertically, with zero Engine drag or inertia observations. The held trace emitted start, nine moves, and end. It produced nine Engine drag and 52–54 inertia observations, with no native cancellation. Actual held travel was 279–293 px vertical by 208–218 px horizontal. The corresponding App callbacks received every semantic event.

### Frame observations and browser controls

The existing HUD tracked 62–65 frame samples per quick trace and 124–127 per held trace, with 0 ms blank in all 26 qualifying traces. The usual per-trace p95 was 16.7–16.8 ms. One unsubscribed quick trace reached p95 33.4 ms and p99 149.9 ms. One unsubscribed held trace reached p99 50 ms. Three rotated pages failed during app boot or module fetch before any input and were excluded. The sample and held-travel spread leave the performance comparison inconclusive.

Four separate trusted trials introduced a second finger 137.5–153.8 ms after native `pointercancel`. Each reported two active contacts, continued native scroll for 364 px, and emitted only the original start and cancel. At a 390 × 844 mobile-emulated viewport, a separate trusted pinch on the stock header outside the grid moved the visual viewport from 1× to 5×. A one-finger pan then moved its offsetTop from 7 to 223 px while grid scrollTop stayed 0. The stock unzoomed page had no vertical overflow.

This is an Engine dev Chromium receipt for AC-1, a pilot for AC-2, and the browser arm of AC-3. Physical iOS/Android and a higher-sample comparison at matched realized travel are still due. Engine physical policy remains tracked at neomjs/neo#22402.

### @neo-gpt-euclid - 2026-11-28T04:11:04Z

## Expanded travel-matched Chromium comparison

I repeated the four subscriber arms from the first receipt on the same real 50,000-contributor grid. DevIndex stayed at `2caef1f1feea39b94d686bf34c3fd831432e2d8e`, Engine at `76d743e589440a8381d1875a92b1a346a0a821cc`, and Chromium at 159.0.8154.6. Dataset, viewport, VDom Worker, 1×1 buffers, native policy, and AC power were held fixed. The run completed 28 rotated blocks of four fresh pages, with one quick and one held trusted trace per page. All 112 pages booted and all 224 traces completed.

The comparison pairs within a rotation. It matches scroll origin (200, 0), grid geometry, trusted input and the expected owner. Realized travel must be within 2 px per axis for quick scroll and 10 px per axis for held drag. I fixed these rules before reading the run. Intervals are fixed-seed 95% percentile bootstrap intervals over whole matched rotations, with 10,000 resamples. Each trace contributes its p95 once.

| Stroke | Effect of adding semantic subscription | Matched / attempted | Mean p95 difference | 95% interval |
| --- | --- | ---: | ---: | --- |
| Quick native | Semantic vs neither | 28 / 28 | +0.014 ms | −0.014 to +0.043 ms |
| Quick native | Both vs raw | 28 / 28 | +0.007 ms | −0.014 to +0.029 ms |
| Held Engine | Semantic vs neither | 21 / 28 | −0.005 ms | −0.038 to +0.029 ms |
| Held Engine | Both vs raw | 20 / 28 | −0.005 ms | −0.030 to +0.020 ms |

All four median p95 differences were 0 ms, and 0 ms of blank time was observed in every trace. Seven held pairs in the first contrast and eight in the second exceeded the realized-travel tolerance. Tail estimates remain descriptive at this frame cadence. The matched p95 comparison shows no detectable timing shift on this browser/device/location sample, with no fleet-wide or physical-device inference.

Every trace repeated the same wire counts. Raw forwarding was 9 quick / 12 held regardless of subscriptions. The zero-subscriber controls emitted zero semantic messages. Subscribed arms emitted 2 quick / 11 held, with the quick cancel handed to native scroll and the held end followed by Engine inertia.

The earlier late-contact and outside-grid controls still apply on these pins. The physical iOS/Android arm of AC-3 and the global touch-default decision in neomjs/neo#22402 remain open.

Raw receipt SHA-256 `c26626670c7f1fe0581a2800061595fd88ca71c43acdc80204c7eb9d79117cc4`. Paired result SHA-256 `348939a83ec93bab1cd3071d99a8126a27e098e89beb6f41f053bab41459352e`.


