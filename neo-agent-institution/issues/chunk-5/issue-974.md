---
id: 974
title: The widened-pane Observatory goldens still show the splitter drag's orbit
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T18:29:16Z'
updatedAt: '2026-11-14T19:01:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/974'
author: neo-opus-vega
commentsCount: 0
parentIssue: 312
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-14T19:01:41Z'
---
# The widened-pane Observatory goldens still show the splitter drag's orbit

## Context

`npm run test-goldens` is red on `dev` (`1ae6c221`, lockfile engine `2f38667e68`). `FleetCockpitVisual` › "the side panel is as wide as the operator drags it, between 280 px and half the body, for the session only" fails at `observatory-pane-widened.png` with 14,234 different pixels. Every one of them is in the canvas; the side panel is identical. The failure was first seen during #972's golden battery and re-measured on `1ae6c221`.

The pin reached `2f38667e68` through Dependabot's #968 (2026-11-14 13:40Z). Like every CI job, it runs no Darwin goldens: CI keeps no pixel authority (#911).

Design authority: neomjs/neo#20786 ("require on-canvas press to orbit GraphScene"). A drag whose press started outside the canvas does not orbit the scene.

## The Problem

The arm drags `.fm-observatory-splitter` 200 px to the left with the button held, so the pointer crosses the canvas. Both widened goldens were captured on engine `08dfefc634`, where GraphScene orbited the camera on any held primary button. That drag orbited the scene, and the goldens recorded the orbit. Since neomjs/neo#20786 (`2d557e454e`, between that engine and the pin), only a press that started on the canvas orbits it. The drag no longer turns the camera.

Measured on `1ae6c221`:
- With neomjs/neo#20786 reverted in the installed engine (`src/canvas/Base.mjs` and `src/canvas/GraphScene.mjs`, reverse patch), the arm passes.
- With it restored, the arm fails at 14,234 px.
- Reverting only GraphScene's one-line condition still fails (14,021 px), because Base's per-canvas `pressedButtons` tracking is half of the change.

The light golden is captured right after the dark one in the same arm, so it carries the same orbit. Only one failure shows because the arm stops at the dark assertion.

## The Architectural Reality

- Arm: `test/playwright/visual/FleetCockpitVisual.spec.mjs`, the `drag` helper and the two `toHaveScreenshot` calls (`:1933`, `:1935`).
- Goldens: `test/playwright/visual/__screenshots__/FleetCockpitVisual.spec.mjs/observatory-pane-widened.png` and `-light.png`, last captured in `499339f5` (#916).
- Stamp: `test/playwright/visual/__screenshots__/baseline-inputs.txt`, written by `npm run stamp-visual-baselines` (`buildScripts/checkVisualBaselines.mjs`). Its `engine` line (`08dfefc634`) is the revision of the last stamp that changed a golden.
- Engine: `GraphScene#updateMouseState` orbits on `mouse.pressedButtons & 1`, which `Base` sets only for presses on that canvas.

## The Fix

Re-capture the battery on macOS at the lockfile engine with `npm run test-goldens -- --update-snapshots=all`. That is the capture mode `buildScripts/checkVisualBaselines.mjs` documents: the default mode rewrites only failing goldens and leaves within-tolerance drift stale but green. The two widened goldens change materially. Any other golden the capture rewrites is within-tolerance drift from the same engine move and ships with them, named in the PR. Stage the changes and re-stamp with `npm run stamp-visual-baselines`; the stamp's `engine` line moves to the pin. No source change.

## Acceptance Criteria

- [ ] `observatory-pane-widened.png` and `observatory-pane-widened-light.png` show the unorbited scene, and the side panel is unchanged. The PR lists every other golden the `=all` capture rewrote, with its pixel delta against the old golden.
- [ ] `baseline-inputs.txt` is re-stamped at the lockfile engine, and `npm run check-visual-baselines` passes.
- [ ] `npm run test-goldens` passes in full at the PR head (macOS).

## Out of Scope

- Any change to the arm, the splitter or GraphScene's orbit rule.
- The other goldens, which pass at the pin.
- Gating Dependabot's engine bumps on a Darwin golden run. That gap is a separate defect-note.

## Avoided Traps

- **Capturing this arm only** (Playwright's default mode) would leave any within-tolerance drift from the same engine move stale but green. That is why the owner documents `=all`.
- **Moving the drag off the canvas to keep the old image** would preserve a behavior the engine removed on purpose.

## Related

#972 / #973 (the battery that surfaced it) · neomjs/neo#20786 · #968 (the pin bump) · #911 · #916 · parent #312

Live latest-open + closed sweep: latest 20 open + closed search ("golden widened", "orbit", "observatory-pane-widened"; Out of Scope / Avoided Traps read on #911 and #527) at 18:28Z; 58 open (red band). There is no equivalent: the nearest, #949 / #952, parse the stamp on a CRLF checkout. The A2A claim sweep (last 30, all read states) found no claim on the goldens, Memory Core no equivalent, and my own assignments nothing on this surface. Structure map: N/A (no new file, two existing goldens plus the stamp).

Origin Session ID: 4c350059-f639-4919-899e-bad8011ada4a


## Timeline

- 2026-11-14T18:29:16Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T18:29:18Z @neo-opus-vega added the `bug` label
- 2026-11-14T18:29:18Z @neo-opus-vega added the `agent-os` label
- 2026-11-14T18:29:18Z @neo-opus-vega added the `ai` label
- 2026-11-14T18:29:18Z @neo-opus-vega added the `testing` label
- 2026-11-14T18:29:30Z @neo-opus-vega added parent issue #312
- 2026-11-14T18:45:02Z @neo-opus-vega cross-referenced by PR #975
- 2026-11-14T19:01:41Z @tobiu referenced in commit `ffa032d` - "test(agentos): re-capture the goldens at the pinned engine (#974) (#975)

Engine neo#20786 orbits GraphScene only on a press that started on the
canvas. The widened-pane arm drags the splitter across the canvas with
the button held, so its two goldens, captured on engine 08dfefc634,
carried an orbit the engine no longer makes.

The battery is re-captured with --update-snapshots=all at the lockfile
engine, as checkVisualBaselines.mjs documents. Beside the two widened
goldens, seven accounts goldens now show AgentDefinitions' documented
A-Z order (Phoebe before Sophie) instead of the fixture's insertion
order, and ten move by anti-aliasing only. Two goldens that render a
per-run value keep their old file. The stamp's engine line moves to the
pin."
- 2026-11-14T19:01:42Z @tobiu closed this issue

