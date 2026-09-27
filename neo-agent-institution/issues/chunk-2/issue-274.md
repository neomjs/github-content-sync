---
id: 274
title: The local Neural Link battery fails six specs on dev that no CI job runs
state: OPEN
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-09-27T00:33:08Z'
updatedAt: '2026-09-27T00:42:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/274'
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
---
# The local Neural Link battery fails six specs on dev that no CI job runs

## Context

While validating #270's pin bump, the Neural Link battery (`npm run test-e2e:nl`, 46 specs) failed the same seven specs on the old engine pin `2965d82` and on neo dev `a50ae57ce8`, with identical errors. The runtime root was a Brain checkout, on a private `NEO_NL_PORT`. CI never runs the battery: the Brain contract job only lists it (`test-e2e -- --list`), and the platform goldens are skipped under `CI`. So a spec that drifts stays red for everyone's local runs. #270's validation needed a full baseline run just to tell its own effect apart. Some of these have been retained as observations on #10 since 2026-09-14 (engine pin 9).

## The Problem

A battery that is always red is not a control: a new regression hides among the old reds. Current failures on dev `1acb7ec`:

| Spec | Failure | First read |
|---|---|---|
| `FleetActivityStreamBurstNL` | expects `● streaming`, gets `● streaming · quiet since Aug 22 04:08 PM` | stale expectation: #175 (`7058353`) added the honest "quiet since" for a live feed over old rows, and this fixture's events are from 2026-08-22 |
| `FleetEmptyStateNL` | the CTA's centre sits 2.0078 px from the card region's, against a ≤ 2 px bound | sub-pixel drift, or a real shift; to measure |
| `FleetMemoriesNL` | `memories-summary-dark` golden mismatch | golden drift; to compare |
| `FleetPermanenceMatrixRow4NL` | the perspective writer refuses `activeLayoutId: null` with layouts present on an `activate: false` save | on #10 since 09-14; test or product contract, to decide |
| `FleetTasksPaneNL` | the source-pill selector also matches pills inside section headers | on #10 since 09-14; test selector |
| `FleetCockpitNWindowNL` | `page.waitForURL`: target page closed mid-gesture | to reproduce |

`FleetCockpitLivenessNL` fails too, but PR #265 (Resolves #263) owns that fix, so it is out of scope here.

## The Architectural Reality

- `test/playwright/playwright.config.e2e.nl.mjs` runs every e2e spec that requests the Brain whitebox `neuralLink` fixture. It needs `NEO_AGENTOS_RUNTIME_ROOT`, a Brain checkout with a generated `ai/config.mjs`; the installed package lacks one.
- A spec's expectation must follow the product: where the product changed on purpose (`#175`), the expectation moves; where it did not, the defect gets its own ticket.

## The Fix

For each spec: reproduce, decide stale expectation versus product defect, then fix the expectation or file the defect. Re-record goldens only after comparing old and new.

## Acceptance Criteria

- [ ] AC-1: on dev plus this change, `npm run test-e2e:nl` passes every spec except three with owners: `FleetCockpitLivenessNL` (#265), `FleetPermanenceMatrixRow4NL` (an engine defect, neomjs/neo#19307, passing once an engine pin carries it) and `FleetCockpitNWindowNL` (#275, a cause still to find). The receipt names the runtime root and port.
- [ ] AC-2: each moved expectation cites the product change that moved it; any product defect found has its own ticket, linked here, and its spec stays red until that ticket lands.

## Out of Scope

- Running the battery in CI (its runtime root and platform goldens need their own decision).
- `FleetCockpitBarCompositionNL` (plain e2e, not the battery): its two darwin goldens are stale since the presets left the bar (defect-note 6a209290).

## Related

#10 (the retained observations), #270 (where the baseline was taken), #265.

Live latest-open sweep: the latest 20 open Institution issues at 2026-09-27T00:32Z hold no battery ticket; `gh search` for the spec names finds none. A2A: no claim on the battery in the last 30 messages. Memory Core: the 09-14 pin-9 observation (neo-gpt, on #10) is the only prior record.

Origin Session ID: 6408fcd4-3571-4ec2-8009-b4dae5d18917
Retrieval Hint: "Neural Link battery red on dev FleetTasksPaneNL FleetPermanenceMatrixRow4NL quiet since stale expectation"


## Timeline

- 2026-09-27T00:33:08Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-27T00:33:09Z @neo-opus-grace added the `bug` label
- 2026-09-27T00:33:09Z @neo-opus-grace added the `agent-os` label
- 2026-09-27T00:33:09Z @neo-opus-grace added the `ai` label
- 2026-09-27T00:33:10Z @neo-opus-grace added the `testing` label
- 2026-09-27T00:39:44Z @neo-opus-grace cross-referenced by #19307
- 2026-09-27T00:41:38Z @neo-opus-grace cross-referenced by PR #19308
- 2026-09-27T00:41:58Z @neo-opus-grace cross-referenced by #275
- 2026-09-27T00:46:24Z @neo-opus-grace cross-referenced by PR #276
- 2026-09-27T00:46:58Z @tobiu referenced in commit `d522b20` - "test(agentos): the Neural Link battery follows the product it tests: quiet since, the card region, section provenance, a taller strip (#274)

Four specs expected a product that has since moved on purpose. The live feed over August rows says quiet since (#175). The empty CTA is measured against the box its stylesheet centers it in, which the old controls-to-grid proxy missed by 2.0078 px. Section heads carry the source of a single-source section, and its rows do not (#113). The Memories goldens grow 6 px with the south strip, since the cockpit bar lost its preset row (60 to 44 px)."

