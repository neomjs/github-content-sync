---
id: 654
title: Start fleet launches seats in waves of two
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T13:40:52Z'
updatedAt: '2026-10-10T16:04:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/654'
author: neo-fable-clio
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-10T16:04:59Z'
---
# Start fleet launches seats in waves of two

## Context

The operator's first `Start fleet` (2026-10-10, eight seats, an M5 Max): every seat's `start` intent left at once; five Claude Desktop seats launched within nine seconds (lease `startedAt` 11:30:09–11:30:18Z, three Codex seats beside them), the host sat under a ~30 s burst, and every Claude seat came up without Memory Core and Knowledge Base — the Fleet's launch-admission proof times out while several seats start (Brain #964, the root fix). The same seats started one by one afterwards (11:34–11:37Z, one every 30–60 s) connected all four rows.

Design authority: the operator's fallback proposal of 2026-10-10 ("Start fleet in chunks or one by one"). #964 removes the proof's fragility; this leaf bounds the burst a fleet start puts on the Fleet, the plane and the host, and makes the button usable before #964 lands.

## The Problem

`FleetBatchController.executeStartFleetBatch` (`apps/agentos/view/fleet/cockpit/FleetBatchController.mjs:200–235`) partitions the roster, then sends one `start` intent per eligible record through a single `Promise.all`; the summary slot stays empty until every intent has answered (the adapter's settle-or-timeout bound is 30 s per intent, `FleetLifecycleIntentAdapter.mjs`). Nothing bounds the burst: N preparations and spawns on the Fleet, N readiness probes plus 2N admission proofs on the plane (#964), N harness boots on the host. The stop batch (`:290–325`) has the same shape, but a stop is a signal, not a boot.

## The Architectural Reality

- The cockpit composes per-seat intents and owns the plan and its words (#618 / #643: `FleetStartPlan.partitionFleetStart`, `summarizeFleetStart`, `renderFleetStartSummary`, `describeFleetButton`); the Brain has no fleet-wide verb, by design.
- `summarizeFleetStart` (`apps/agentos/util/FleetStartPlan.mjs:123–155`) reads a missing result as `rejected` ("start rejected") — true after a `Promise.all`, wrong for a batch that is still sending.
- The adapter's answer is settle-or-timeout within 30 s; a timeout carries `settlement`, which the batch already observes to update the summary late.
- Unit coverage: `test/playwright/unit/apps/agentos/view/fleet/cockpit/startPlan.spec.mjs` (the pure half).

## The Fix

1. `FleetStartPlan.waves(records, size)` — pure: consecutive groups of `size` in roster order; `size` below 1 or not an integer reads as 1 (one by one, the safe direction); an empty list gives no wave.
2. `executeStartFleetBatch` iterates the waves: one wave's intents in parallel, the next wave after the previous wave's answers (the adapter's honesty bound keeps each wave finite); between waves the batch fence holds (`startFleetBatch` token, `isDestroyed`, the bridge profile) — a retired batch sends no further wave. Results stay index-aligned with `plan.eligible`; the settlement handlers and `refreshRosterOnSettle` are unchanged.
3. A running summary between waves: `summarizeFleetStart` reads a missing result as `pending` (a count, plus `<agentId>: start pending` detail lines); `renderFleetStartSummary` prints `N started · P pending · M excluded` while `pending > 0`; the final line is unchanged.
4. `startFleetWaveSize` — a documented static config on `FleetBatchController`, default `2` (two concurrent redemption proofs plus the next wave's readiness probes stay well under #964's 10 s bound on an idle plane; `1` is one by one).

The stop batch keeps its single wave.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetStartPlan.waves` (new, `util/FleetStartPlan.mjs`) | this ticket | consecutive groups of `size`; `size` < 1 or non-integer → 1; empty → `[]` | — | JSDoc | unit (AC-1) |
| `FleetStartPlan.summarizeFleetStart` (`:123`) | #618 | a missing result counts as `pending` (new field), never as rejected | existing results unchanged | JSDoc | unit (AC-2) |
| `FleetStartPlan.renderFleetStartSummary` | #618 | `· P pending` in the text while pending > 0; pending detail lines | unchanged when pending = 0 | JSDoc | unit (AC-2) |
| `FleetBatchController.startFleetWaveSize` (new config) | this ticket | default 2; read once per batch | — | config JSDoc | unit (AC-3) |
| `FleetBatchController.executeStartFleetBatch` (`:200`) | #643 | waves, running summary, fence between waves | a single wave when `size ≥ eligible.length` (today's behavior) | method JSDoc | unit (AC-3) + installed (AC-4) |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1 — unit: `waves` splits 8 records into `[2,2,2,2]` at size 2 and `[3,3,2]` at 3; size 1 gives singletons; 0, `NaN` and `1.5` read as 1; an empty list gives `[]`; roster order is kept.
- [ ] AC-2 — unit: a summary over a partial result array reports `pending` and renders `1 started · 3 pending · 1 excluded` with the pending seats in the detail; the existing summary tests pass unchanged.
- [ ] AC-3 — unit (a controller with a fake `requestFleetLifecycle` recording call order and timing): wave k+1 is sent only after wave k's answers; a retired batch token stops the loop with no further intent; the final summary equals today's for the same answers.
- [ ] AC-4 — installed witness (post-merge, the operator or a non-author seat): `Start fleet` with at least five Claude Desktop seats down — the leases' `startedAt` show the waves, the summary slot counts `pending` down, and with #964 still unmerged every seat's four MCP rows connect; recorded on Institution #12 with the candidate's tuple.

## Out of Scope

- The root fix of the admission proof (Brain #964).
- The stop batch, a Brain-side `startFleet` verb (the cockpit composes per-seat intents, #618 / #643), an adaptive wave size from plane health.

## Avoided Traps

- Gating the next wave on `settlement` instead of the answer: one Start that never settles would stall the whole fleet; the adapter's honesty bound keeps a wave finite.
- Rendering a missing result as `rejected` mid-batch (today's fold): `pending` is a distinct word, so the running line never claims a refusal that has not happened.
- Moving the policy into the Brain: the burst is a cockpit batch's property; the Brain keeps answering one intent at a time.

## Related

Brain #964 (the admission proof); Institution #618 / #643 (the fleet button and its plan), #477 (the cockpit's words), #12 (installed acceptance), #633 (Skip and Cancel start witness).

Sweeps: live latest-20 open Institution issues read 2026-10-10 13:37:04Z — no equivalent (#633 witnesses Skip / Cancel on a Fleet Start; #618 closed by #643); A2A last 30 at 13:4xZ — claims on the dock-polish leaves (Vega #19544, Grace #19548, Mnemosyne #19549), Sophie's #964 intake, none on the fleet batch; Memory Core — the 2026-10-09 planner input and #618's filing decided the plan-driven button, nothing on pacing; own assignments (#505 #507 #351) — none on this surface; structure map: an Institution app leaf, owning folder `apps/agentos/view/fleet/cockpit` + `apps/agentos/util`, no new file.

Retrieval Hint: "Start fleet waves startFleetWaveSize pending summary executeStartFleetBatch"

Origin Session ID: f45d36fd-6e77-4c89-bd56-dd49d95b5b0a

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session f45d36fd-6e77-4c89-bd56-dd49d95b5b0a

## Timeline

- 2026-10-10T13:40:52Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T13:40:53Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T13:40:53Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T13:40:54Z @neo-fable-clio added the `ai` label
- 2026-10-10T13:41:05Z @neo-fable-clio added parent issue #477
- 2026-10-10T13:56:23Z @neo-fable-clio cross-referenced by #655
- 2026-10-10T13:58:33Z @neo-fable-clio cross-referenced by PR #656
- 2026-10-10T14:07:27Z @neo-fable-clio cross-referenced by #658
- 2026-10-10T15:05:11Z @neo-fable-clio referenced in commit `918e8c3` - "feat(agentos): Start fleet launches seats in waves of two (#654)

One fleet-start press sent every eligible seat's start intent at once; the
first real Start fleet left every Claude Desktop seat without Memory Core and
Knowledge Base (Brain #964: the launch-admission proof times out under
concurrent starts) and the host under a ~30 s burst. The batch now sends its
seats in waves — FleetStartPlan.waves, startFleetWaveSize = 2 on
FleetBatchController, the next wave once the previous one answered — and the
chrome line counts the unsent seats as pending between waves instead of
folding a missing result into "rejected". A retired batch sends no further
wave. The stop batch keeps its single wave."
- 2026-10-10T15:54:45Z @neo-fable-clio referenced in commit `d3a00a5` - "fix(agentos): a wave reads its seats before it leaves, a retired roster ends the batch, late answers land at once (#654)

Sophie's falsifier against the real controller chain found three contract
holes in the wave batch: a seat its own card started while an earlier wave was
held was sent a second time; a roster retired and re-bound to the same profile
id let the old batch keep sending; an earlier wave's late answers reached the
running line only after the next wave answered. Each wave now re-partitions
its seats as they are when it leaves (a superseded seat keeps its slot with
the partition's reason), onRosterRetired drops the batch token, and the
settlement watchers attach per wave. Three mirror tests in fleetControl.spec."
- 2026-10-10T15:55:08Z @neo-fable-clio referenced in commit `b838e9f` - "test(agentos): the wave fixture's comment describes the harness, not the review (#654)"
- 2026-10-10T16:04:59Z @tobiu referenced in commit `52925d8` - "feat(agentos): Start fleet launches seats in waves of two (#654) (#656)

* feat(agentos): Start fleet launches seats in waves of two (#654)

One fleet-start press sent every eligible seat's start intent at once; the
first real Start fleet left every Claude Desktop seat without Memory Core and
Knowledge Base (Brain #964: the launch-admission proof times out under
concurrent starts) and the host under a ~30 s burst. The batch now sends its
seats in waves — FleetStartPlan.waves, startFleetWaveSize = 2 on
FleetBatchController, the next wave once the previous one answered — and the
chrome line counts the unsent seats as pending between waves instead of
folding a missing result into "rejected". A retired batch sends no further
wave. The stop batch keeps its single wave.

* fix(agentos): a wave reads its seats before it leaves, a retired roster ends the batch, late answers land at once (#654)

Sophie's falsifier against the real controller chain found three contract
holes in the wave batch: a seat its own card started while an earlier wave was
held was sent a second time; a roster retired and re-bound to the same profile
id let the old batch keep sending; an earlier wave's late answers reached the
running line only after the next wave answered. Each wave now re-partitions
its seats as they are when it leaves (a superseded seat keeps its slot with
the partition's reason), onRosterRetired drops the batch token, and the
settlement watchers attach per wave. Three mirror tests in fleetControl.spec.

* test(agentos): the wave fixture's comment describes the harness, not the review (#654)"
- 2026-10-10T16:04:59Z @tobiu closed this issue
- 2026-10-10T18:31:52Z @neo-gpt-sophie cross-referenced by #12

