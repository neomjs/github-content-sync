---
id: 3
title: Fleet list conversions regain their visual contracts
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - design
  - regression
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-08-27T08:54:38Z'
updatedAt: '2026-08-28T13:41:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/3'
author: tobiu
commentsCount: 2
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1 The Fleet Manager arrives with its own test suite'
blocking: []
closedAt: '2026-08-28T13:41:16Z'
---
# Fleet list conversions regain their visual contracts

## Context

The Fleet Manager extraction preserves the existing visual goldens byte-for-byte. During the move, the operator identified substantial design regressions after two deliberate topology changes:

- the roster became an animated `Neo.list.Component` surface;
- the activity stream became `Neo.list.Buffered`.

Live latest-open sweep: checked immediately before creation at 2026-08-27T08:54:37.066Z; no equivalent Institution ticket existed.

## The Problem

The topology changes are valuable, but the current AgentCard/grid and buffered activity render no longer consistently satisfy their established density, containment, and visual contracts. Refreshing snapshots during extraction would hide that signal; reverting the list primitives would discard the intended architecture.

The receiver therefore moves the failing evidence unchanged and defers the repair here. The Fleet visual script is intentionally absent from Ubuntu CI, leaving one spec plus six Darwin goldens with zero CI reach; this ticket also owns a platform-honest drift signal.

## The Architectural Reality

- `apps/agentos/view/fleet/roster/List.mjs` owns the animated component list and selection model.
- `apps/agentos/view/fleet/roster/card/Container.mjs` owns card anatomy; the list owns seating, movement, and scroll geometry.
- `apps/agentos/view/fleet/activity/Container.mjs` owns a fixed-height `Neo.list.Buffered` row pool.
- `test/playwright/e2e/agentos/AgentCardSynthesisRenderNL.spec.mjs` and `test/playwright/visual/FleetCockpitVisual.spec.mjs` are the design receipts; their goldens are review surfaces, not auto-healing artifacts.

The list conversions entered Neo in commits `4699d2207f` and `68505dfc37`.

Decision Record impact: none.

## The Fix

1. Run the unchanged AgentCard and Fleet visual suites red-first and classify each delta as list geometry, card/row SCSS, stale golden, or unrelated product drift.
2. Repair the owning list/card/activity styles and sizing contracts without reverting animation, selection, or buffering.
3. Re-run the semantic geometry/accessibility guards before considering any golden refresh.
4. Refresh only images whose new pixels are an explicitly reviewed design outcome.
5. Add a platform-honest visibility mechanism so Darwin goldens cannot rot silently; never promote Ubuntu rendering as Darwin visual evidence.

## Acceptance Criteria

- [ ] Red-first receipts identify the exact AgentCard/grid and activity-stream failures before mutation.
- [ ] Animated roster cards satisfy their existing narrow/boundary/regular/roomy containment guards.
- [ ] Buffered activity rows remain readable and non-overlapping across the existing viewport contracts.
- [ ] Sorting, selection, recycling, scroll ownership, and new-event behavior remain intact.
- [ ] Both the semantic guards and the reviewed visual suites pass.
- [ ] Every refreshed golden is explained by an intentional design delta; unrelated baselines remain byte-identical.
- [ ] The one visual spec plus six Darwin goldens have a durable drift signal without false Ubuntu pixel authority.

## Out of Scope

- Reverting the roster to a hand-built card array.
- Reverting the activity stream to an unbuffered list.
- Provider-validation staleness; #2 owns that narrow repair.
- Broad Fleet feature work unrelated to rendering.

## Related

Related: #1

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: "Institution AgentCard animated list BufferedList visual baseline regressions"

Authored by Emmy (GPT-5.6 Sol Ultra, Codex). Session d39e8182-295f-418a-82cd-a96be9c08e4f.



## Timeline

- 2026-08-27T08:54:39Z @tobiu added the `bug` label
- 2026-08-27T08:54:39Z @tobiu added the `agent-os` label
- 2026-08-27T08:54:39Z @tobiu added the `ai` label
- 2026-08-27T08:54:40Z @tobiu added the `design` label
- 2026-08-27T08:54:41Z @tobiu added the `regression` label
- 2026-08-27T08:54:41Z @tobiu added the `testing` label
- 2026-08-27T08:54:48Z @neo-gpt-emmy marked this issue as being blocked by #1
- 2026-08-27T08:55:08Z @tobiu cross-referenced by #1
- 2026-08-27T09:04:00Z @tobiu cross-referenced by PR #5
- 2026-08-27T09:09:56Z @neo-gpt-emmy cross-referenced by PR #6
- 2026-08-27T11:12:44Z @neo-gpt cross-referenced by #25
- 2026-08-27T11:33:39Z @neo-opus-vega cross-referenced by PR #26
### @neo-gpt - 2026-08-27T11:37:57Z

## README hero is now a second golden consumer

Agent Institution PR #26 uses `test/playwright/visual/__screenshots__/FleetCockpitVisual.spec.mjs/cockpit-default-shell.png` as the public README hero. The caption identifies it as the offline/static visual golden, so provenance is honest; the lifecycle coupling is now public-facing.

When #3 classifies or re-baselines that golden, validate the README rendering as a second consumer and keep the planned durable drift signal aware that `test-visual` remains outside `ci.yml`. This does not block PR #26 and adds no new visual-repair scope; #3 already owns the exact golden/CI contract.

- 2026-08-28T09:54:56Z @neo-fable-clio cross-referenced by #29
- 2026-08-28T09:55:05Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-08-28T09:55:21Z

**Lane claim — taking this** (self-assigned, 2026-08-28). The two topology changes are my own pre-split work (commits `4699d2207f` + `68505dfc37`), so the design debt they carried across the extraction lands back where it belongs.

**Fresh operator evidence (2026-08-28, screenshot of the live activity stream)** — extending the ticket's density/containment framing with today's observed symptom set:

1. **Row height:** the `list.Buffered` rows render at roughly double the before-state height — the row pool inherits a default `rowHeight` instead of the stream's compact contract; density is the first casualty.
2. **Mixed vertical anchoring inside one row:** timestamps sit top-anchored while chip/sender/message content centers vertically — two anchoring models in the same row.
3. **Focus treatment:** the focused row renders a raw full-row outline (default styling), not a designed focus state for this surface.
4. **Meta header collapse:** the header region ("LIVE ACTIVITY · 6 retained · sample · live feed pending") renders as a three-line CENTERED block floating above the rows, where the before-state had a single left-aligned header bar with the sample/live qualifier right-aligned.
5. **Sender label alignment:** the sender (`neo-opus-vega` …) floats between the type chip and the message text without a declared alignment relationship to either.
6. **Timestamp format width:** long-form timestamps (`Jul 5 12:52 PM`) consume noticeably more row width than the before-state short form — worth deciding deliberately as part of the density contract, not inheriting from data format.

Symptoms 1–3 confirm the ticket's core claim; 4–6 are additions I will fold into the same visual-contract repair (they share the row/header anatomy — splitting them would violate the one-lane bundling default).

Anchors from today's read: the activity pane mounts via the cockpit pane factory (`cockpit/Container.mjs`, case `'activity-stream'`) onto `activity/Container.mjs`'s fixed-height `Neo.list.Buffered` row pool; the roster half of this ticket stays in scope per the body. The checked-in goldens remain the review surface — they get updated only as witnessed repair evidence, never refreshed to hide the signal.

Branch + PR to follow on this repository. Retrieval hint: "activity stream buffered list visual contract rowHeight anchoring focus header".


- 2026-08-28T10:06:27Z @neo-fable-clio cross-referenced by #30
- 2026-08-28T11:11:09Z @tobiu cross-referenced by PR #31
- 2026-08-28T11:18:57Z @neo-fable-clio referenced in commit `e7cd0d4` - "fix(agentos): parity suite gates honestly on the post-split twin seam (#3)"
- 2026-08-28T11:19:23Z @neo-fable-clio cross-referenced by PR #32
- 2026-08-28T11:42:40Z @neo-fable-clio cross-referenced by PR #33
- 2026-08-28T12:49:12Z @tobiu referenced in commit `c21f2cb` - "fix(agentos): sub-narrow card grammar + drift-gate golden coverage + exact matrix poll (#3)"
- 2026-08-28T13:41:16Z @tobiu referenced in commit `7f3f33a` - "Merge pull request #32 from neomjs/agent/3-activity-stream-visual-contracts

feat(agentos): regain the fleet visual contracts — consumer scss build, containment, drift gate (#3)"
- 2026-08-28T13:41:17Z @tobiu closed this issue
- 2026-08-28T14:10:14Z @neo-fable-clio cross-referenced by PR #34
- 2026-08-28T15:41:30Z @neo-fable-clio cross-referenced by PR #35
- 2026-09-02T13:15:17Z @neo-fable-clio cross-referenced by PR #77

