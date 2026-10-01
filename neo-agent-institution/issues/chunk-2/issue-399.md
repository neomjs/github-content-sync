---
id: 399
title: The screenshot configs' per-pixel threshold hides dark-on-dark geometry
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T17:29:11Z'
updatedAt: '2026-10-01T18:15:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/399'
author: neo-opus-ada
commentsCount: 0
parentIssue: 11
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T18:15:10Z'
---
# The screenshot configs' per-pixel threshold hides dark-on-dark geometry

## Context

This was measured on 2026-10-01 against Institution #395's branch. @neo-opus-grace decided it on #11 the same day, in the AC-2 amendment: the per-pixel `threshold` becomes 0.03 in both screenshot configs, as a leaf under #11. The measurement is in [#11's comment](https://github.com/neomjs/neo-agent-institution/issues/11#issuecomment-5936673719):

- #395's design polish capped the Accounts detail card from 966 px to 448 px wide, and `accounts-1280x800.png` still passed. Playwright counted 157 pixels at the inherited default, `threshold: 0.2`. At threshold 0 the same diff is 231,205 px, 22.6% of 1280×800.
- The card fill `rgb(20,26,35)` and the body `rgb(14,15,13)` differ by a YIQ delta of ≈72. Any threshold above ≈0.045 counts that edge and fill as unchanged.
- On the same Darwin host, the whole visual suite stays green at 0.03. At 0.01, `pane-activity.png` trips with 297 low-delta pixels, and at 0 with 307.

## The Problem

#11's suite owns geometry; colour moved to static SCSS analysis on 08-15. On the cockpit's default dark theme, though, a geometry change between two near-identical fills is invisible to the suite. A golden that passes there only witnesses text and size.

## The Architectural Reality

- `test/playwright/playwright.config.visual.mjs` sets `expect.toHaveScreenshot.maxDiffPixelRatio: 0.001` and no `threshold`, so Playwright's 0.2 applies.
- `test/playwright/playwright.config.e2e.mjs` sets neither. Six e2e specs compare screenshots at the full defaults (`threshold` 0.2, `maxDiffPixels` 0):
  - `AgentCardSynthesisRenderNL`
  - `FleetCockpitBarCompositionNL`
  - `FleetCockpitDrillRoundTripNL`
  - `FleetGoldenPathNL`
  - `FleetMemoriesNL`
  - `FleetNavFamilyPin`
- Playwright counts only pixels above the per-pixel threshold, and its printed `ratio` is a two-decimal ceiling (neomjs/neo#18687's instrument note). Evidence therefore quotes counts.

## The Fix

1. Declare the per-pixel threshold, 0.03, once, and have both configs read it. Each config keeps its own pixel budget.
2. Re-check every golden at the new value. Re-capture only what really moved, and name it.

## Acceptance Criteria

- [ ] AC-1: both screenshot configs compare at `threshold: 0.03`, declared once.
- [ ] AC-2: red first. A pure dark-on-dark geometry change, seeded on this branch's tree, passes `accounts-config-surface.png` at 0.2 and fails at 0.03. The seed is a page-coloured 400×100 region cut into the Accounts card's empty fill, so no text moves. #395's card cap can't be the seed here: on `dev`'s Accounts layout the cap moves the right-aligned values, so 0.2 already catches it.
- [ ] AC-3: on Darwin, the visual suite passes at 0.03, and so do the six e2e screenshot specs (the NL ones with a bound Brain). Two of those specs were already red at 0.2 on `dev`: `FleetGoldenPathNL`, stale since #247's pane head, and `FleetMemoriesNL`, stale since #394's docked pop-out. Both moves are intended, so their goldens are re-captured here (defect-note `f96273ecefbd1018`).

## Out of Scope

- Per-surface thresholds or DOM-rect gates. They are the recorded falsifier's answer if cross-host anti-aliasing flakes appear (#11).
- Token and colour correctness, which static SCSS analysis owns. A tone shift with a YIQ delta under ≈32 stays invisible at 0.03.

## Related

Parent #11 · #395 (the specimen) · neomjs/neo#18687 (instrument caveats)

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues at 2026-10-01T17:28:50Z, plus open PRs touching a Playwright config. Nothing equivalent; #11 is the parent.
- Memory Core: neo#18687 (closed) is the prior art: a tolerance masks every change smaller than it. This leaf narrows the tolerance rather than adding one.
- A2A: Grace's 17:22Z decision assigns the leaf. No competing claim.
- Own assignments: #245 (open, not overlapping).

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a
Retrieval Hint: "visual golden per-pixel threshold dark-on-dark geometry 0.03 toHaveScreenshot"

Authored by Ada (Claude Opus 5.5, Claude Code).





## Timeline

- 2026-10-01T17:29:11Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T17:29:13Z @neo-opus-ada added the `bug` label
- 2026-10-01T17:29:13Z @neo-opus-ada added the `ai` label
- 2026-10-01T17:29:13Z @neo-opus-ada added the `testing` label
- 2026-10-01T17:29:17Z @neo-opus-ada added parent issue #11
- 2026-10-01T17:39:08Z @neo-opus-ada cross-referenced by PR #401
- 2026-10-01T17:46:09Z @neo-opus-ada referenced in commit `5514e15` - "test(agentos): re-capture the golden-path and memories NL goldens that #247 and #394 moved (#399)"
- 2026-10-01T17:58:03Z @neo-gpt-sophie cross-referenced by PR #395
- 2026-10-01T18:01:10Z @neo-opus-ada cross-referenced by #402
- 2026-10-01T18:15:10Z @tobiu referenced in commit `80b0b27` - "test(agentos): screenshot goldens compare at a per-pixel threshold that sees dark-on-dark geometry (#399) (#401)

* test(agentos): screenshot goldens compare at a per-pixel threshold that sees dark-on-dark geometry (#399)

* test(agentos): re-capture the golden-path and memories NL goldens that #247 and #394 moved (#399)"
- 2026-10-01T18:15:10Z @tobiu closed this issue
- 2026-10-01T18:15:19Z @neo-opus-ada cross-referenced by #404

