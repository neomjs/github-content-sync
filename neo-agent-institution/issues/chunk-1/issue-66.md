---
id: 66
title: 'The cockpit''s Neural Link dock witnesses are dark: nine rotted through the August rebuilds, two read the retired zone form'
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - regression
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-01T20:57:20Z'
updatedAt: '2026-09-01T22:51:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/66'
author: neo-fable-clio
commentsCount: 0
parentIssue: 10
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-01T22:51:05Z'
---
# The cockpit's Neural Link dock witnesses are dark: nine rotted through the August rebuilds, two read the retired zone form

## Context

The cockpit's dock behaviour is proven by the `FleetCockpit*NL.spec.mjs` battery under `test/playwright/e2e/agentos/` — seventeen Neural Link journeys (drill, pop-out, tear-out, rail reveal, perspective presets, liveness, N-window, viewer wake). None of them runs in CI: the isolated job runs only the non-NL specs, and the cross-repository job merely `--list`s the suite. Measured on 2026-09-01 by running the battery locally with the Brain runtime root (`NEO_AGENTOS_RUNTIME_ROOT=<brain checkout> npx playwright test -c test/playwright/playwright.config.e2e.mjs agentos/FleetCockpit --workers=1`):

| head | Engine pin | result |
|---|---|---|
| `dev@f97187a` | `17b59aad` (pre-cut) | **9 failed / 8 passed** |
| PR #65 head `cdc737d` | `dev@0659b0e42d` | 11 failed / 6 passed (the 9 above + 2 new) |

Nine witnesses have been red on `dev` since the August rebuilds, unnoticed because nothing runs them. The battery is the product's proof surface for docking; a dark battery certifies nothing.

## The Problem

| Spec | Failure | Rotted by | Fix shape |
|---|---|---|---|
| `FleetCockpitAutoHideRailNL` | expects 4 rail tabs to survive pinning the detail; the document carries 4 rail items, so 3 remain | the seeded document (tasks moved to the south strip before the split) | derive the expected count from the committed document, not a literal |
| `FleetCockpitBarCompositionNL` (800 + 520) | `toHaveScreenshot` goldens show the pre-#58 banner vocabulary ("Fleet server offline — …", "wake: wake stream disconn…") | #58 (status-word pills) never re-captured them | re-capture; **and** the 800px actual clips the pill to "fleet offl:" — a real bar-composition defect for the #23 collapse order, not a golden problem |
| `FleetCockpitDrillNL` | `.fm-agent-detail .fm-detail-sources` not found | #60 (agent-detail rail IA replaced the sources block with the state ledger) | assert the ledger the pane renders now |
| `FleetCockpitDrillRoundTripNL` | same #60 surface (`.fm-detail-sources`, pop-out toggle moved to the tab-seam action) | #60 | re-target the drill leg |
| `FleetCockpitPopOutNL` | `.fm-detail-window-toggle` text assertion | #60 (the toggle became the detail strip's tab-seam action) | drive the seam action, keep the round-trip contract |
| `FleetCockpitLivenessNL`, `FleetCockpitNWindowNL`, `FleetCockpitViewerWakeNL` | `Method not found: loadRoster / stopLiveness on instance neo-fm-fleet-cockpit-1` | #50 (both moved to `LivenessController`) | call the controller |
| `FleetCockpitDockNL` #1 (**new on PR #65**) | `zones.center` expected `'primary-split'`, received `{nodeId: 'primary-split'}` | the final Engine model reads nested zone descriptors (Engine #17837; consumer adoption #39) | assert the descriptor (patch ready) |
| `FleetCockpitDockNL` #2 (**new on PR #65**) | the Review preset's materialized detail pane stays `hidden` for 10s in **headless** Chromium; passes headed on the same head; passes headless on the old pin | Engine `dev@0659b0e42d` — the projection Reconciler stages tab chrome with `hideMode: 'visibility'` and un-hides on the FLIP settle; in headless the settle does not land | Engine defect-note filed via A2A; until it lands, run this witness headed or gate it on the Engine fix |

Passing on both heads: DockGeometry, FocusInvariant, Kinetic, RailDrawerErgonomics, TearOut ×2.

## The Architectural Reality

- The witnesses are the cockpit's only proof of the dock choreography the operator ships; the August rebuilds (#50 core, #58 bar, #60 detail rail) moved the surfaces the specs targeted and nobody re-ran the battery, because no job does.
- The `check-visual-baselines` stamp covers the Darwin goldens' inputs, not the NL snapshot goldens or the NL specs — a stale NL golden is invisible to the freshness gate.
- The two PR #65 items are consumer-side truth (the descriptor) and an Engine-side motion hold; the repair keeps them apart.

## The Fix

One repair PR, witness by witness, against the current product (never the product against the witnesses): the rows above, the bar goldens re-captured on Darwin, the DockNL descriptor assertion, and the headless-hold witness marked with its Engine dependency. Add a documented `test-e2e:nl` script that runs the battery with the Brain runtime root so the receipt is one command. The 800px pill clipping goes back to the #23 collapse order as its own design fix (the golden must not be captured over a defect). CI wiring stays with #64.

## Acceptance Criteria

- [ ] The seventeen `FleetCockpit*NL` journeys pass headed on `dev` at the current Engine pin, except the one gated on the Engine headless hold, which names the Engine ticket or note it waits for.
- [ ] Each repaired assertion targets the current surface (controller methods, the detail ledger, the tab-seam action, the nested descriptor) — no compatibility shim in the app.
- [ ] The bar-composition goldens are re-captured after the 800px pill clipping is fixed under #23, never over it.
- [ ] `test-e2e:nl` runs the battery with the Brain runtime root and is documented in the README's test section.
- [ ] The visual-baseline stamp is refreshed in the same PR.

## Out of Scope

- Wiring the NL battery into CI (#64's baseline consumption).
- The Engine headless visibility hold itself (Engine-owned; defect-noted).
- New witnesses for the default action rail, reload and header pop-out (the FM DockLayouts leverage cuts under #10 carry their own).

## Related

Related: #10 (parent) · #23 (the 800px pill clipping) · #24 · #64 · #11 (visual harness, adjacent) · PR #65 (the two new rows) · neomjs/neo#17836

Live latest-open sweep: checked all 23 open Institution issues at 2026-09-01T20:56Z — no witness-repair ticket exists (#11 is the pixel-golden harness, #63 a chip regression). A2A latest-30 all-status sweep at 20:33Z: no competing lane claim. MC sweep: `query_raw_memories("FleetCockpit NL witnesses rot loadRoster stopLiveness fm-detail-sources")` — no prior decision; the rebuild sessions (#50/#58/#60) record no battery run. Own-assignment sweep: 0 open Institution issues assigned to me.

unowned-rationale: filed unassigned on purpose — it is the residual owner for PR #65 and the honest ledger of my own August rebuilds' witness debt; I pick it up after the FM DockLayouts leverage cuts unless a peer wants the battery first.

Origin Session ID: 8c42ad77-48b0-448a-992a-219df09cd4c6

Retrieval Hint: `query_raw_memories("FleetCockpit Neural Link battery dark nine rotted specs headless visibility hold DockNL")`

📜 Clio

## Timeline

- 2026-09-01T20:57:22Z @neo-fable-clio added the `bug` label
- 2026-09-01T20:57:22Z @neo-fable-clio added the `agent-os` label
- 2026-09-01T20:57:22Z @neo-fable-clio added the `ai` label
- 2026-09-01T20:57:22Z @neo-fable-clio added the `regression` label
- 2026-09-01T20:57:23Z @neo-fable-clio added the `testing` label
- 2026-09-01T20:58:17Z @neo-fable-clio cross-referenced by PR #65
- 2026-09-01T21:00:06Z @neo-fable-clio cross-referenced by #17836
- 2026-09-01T21:00:07Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-01T21:41:37Z @neo-fable-clio referenced in commit `66f69bd` - "test(agentos): the Review-preset hold is the Engine race under headed runs too (#66)

The inline note claimed the FLIP-settle hold is headless-only; the final full headed battery of the night reproduced it once in three runs. The note now says what was measured so a red at this step reads as the Engine race, not the journey."
- 2026-09-01T21:42:14Z @neo-fable-clio cross-referenced by PR #70
- 2026-09-01T21:52:35Z @neo-fable-clio referenced in commit `8a79697` - "test(agentos): the keyboard-a11y witness imports the helper it calls (#66)

FleetGridKeyboardA11y called loadAgentOsModule without importing it, so it died on a ReferenceError before its first assertion. With the import restored it reaches the roster and fails honestly at its first keyboard read (ArrowDown leaves native focus on the first list item) — a pointer for triage, ledgered on the PR, not masked."
- 2026-09-01T22:04:18Z @neo-opus-ada cross-referenced by PR #71
- 2026-09-01T22:35:25Z @neo-fable-clio referenced in commit `d5b02fd` - "chore(test): merge dev and refresh the visual baseline stamp (#66)"
- 2026-09-01T22:42:34Z @neo-opus-grace cross-referenced by PR #72
- 2026-09-01T22:51:06Z @tobiu referenced in commit `0b7542b` - "Merge pull request #70 from neomjs/agent/66-nl-witness-repair

test(agentos): repair the Neural Link dock witnesses for the rebuilt cockpit (#66)"
- 2026-09-01T22:51:06Z @tobiu closed this issue
- 2026-09-01T22:53:46Z @neo-fable-clio cross-referenced by #73
- 2026-09-01T23:30:31Z @neo-fable-clio cross-referenced by PR #75
- 2026-09-02T14:46:01Z @neo-fable-clio cross-referenced by #81
- 2026-09-02T14:52:33Z @neo-fable-clio cross-referenced by PR #82
- 2026-09-02T15:28:45Z @neo-fable-clio cross-referenced by PR #83
- 2026-09-04T09:27:32Z @neo-fable-clio cross-referenced by #90
- 2026-09-04T09:55:22Z @neo-fable-clio cross-referenced by PR #91
- 2026-09-04T10:08:33Z @neo-fable-clio cross-referenced by PR #93
- 2026-09-04T11:21:52Z @neo-fable-clio cross-referenced by PR #95
- 2026-09-04T14:23:43Z @neo-fable-clio cross-referenced by PR #102
- 2026-09-04T14:57:52Z @neo-fable-clio cross-referenced by #103

