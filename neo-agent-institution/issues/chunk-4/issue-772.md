---
id: 772
title: Show both Stop fleet and Start fleet when the active fleet is split
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T20:30:12Z'
updatedAt: '2026-10-21T09:38:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/772'
author: neo-gemini-maryam
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
closedAt: '2026-10-21T09:38:38Z'
milestone: FM v1
---
# Show both Stop fleet and Start fleet when the active fleet is split

## Context

When the active fleet is partially online — for example, `8 started · 3 excluded` with `2` active seats offline, or after 5 Claude harnesses close on an auto-update / session end while 5 Gemini and GPT seats remain online — the cockpit control bar currently renders only `▶ Start fleet · 2` (or `▶ Start fleet · 5`) and hides `Stop fleet` completely. When the operator wants to wind down ("let's call it a day") or pause the running seats before an update, `Stop fleet` is unreachable from the top bar and the operator must stop each running seat's roster card one by one.

Design authority: `#477` ("Every cockpit surface names its state with a reason and a next step — row 2 of FM v1"), `#618` / PR `#643` ("The fleet button reads the plan it would run, and offers Stop fleet"), and operator UX direction (2026-10-20).

## The Problem

At `neomjs/neo-agent-institution` `dev` `17c3d6deffc8e8e2d5875c99123b47d79e453f22`, `FleetStartPlan.describeFleetButton(records)` (`apps/agentos/util/FleetStartPlan.mjs:249-278`) forces a mutually exclusive priority waterfall onto a single toolbar button (`reference: 'fleet-button'` in `apps/agentos/view/fleet/cockpit/ControlToolbar.mjs:150-156`):

- If `start.eligible.length > 0`, it returns `{action: 'start', count: start.eligible.length, text: 'Start fleet · M'}` and suppresses `Stop fleet · N` regardless of how many active seats are currently running (`stop.eligible.length`).
- `Stop fleet · N` is offered only when `start.eligible.length === 0 && stop.eligible.length > 0`.

In a split fleet (`start.eligible.length > 0` AND `stop.eligible.length > 0`), both batch intents are genuinely live and orthogonal: topping up the offline active seats (`Start fleet · M`) vs shutting down the running active seats (`Stop fleet · N`).

## The Architectural Reality

- `apps/agentos/util/FleetStartPlan.mjs:74-116, 203-278` — `partitionFleetStart(records)` and `partitionFleetStop(records)` already compute the exact disjoint eligible sets (`state === 'off'` vs `state !== 'off'` across active, wired, non-pending, non-timed-out seats). `describeFleetButton(records)` collapses both partitions into one `{action, count, text, title}` descriptor.
- `apps/agentos/view/fleet/cockpit/ControlToolbar.mjs:146-156` — declares a single `Button` (`cls: ['fm-fleet-start']`, `reference: 'fleet-button'`, `handler: 'onFleetButton'`).
- `apps/agentos/view/fleet/cockpit/FleetBatchController.mjs:71-145, 199-328` — owns `onFleetButton()`, `renderFleetButton()`, `onStartFleet()`, and the two-press `onStopFleet()` (`stopFleetArmed`, `stopFleetArmTimer`, `isStopArmCurrent`, `disarmStopFleet`). `onStartFleet()` (`line 202`) already calls `me.disarmStopFleet()`.
- `apps/agentos/view/fleet/cockpit/Controller.mjs:232-247` — `onAgentLifecycleIntent` drives 1-click per-card `stop` and calls `this.disarmStopFleet()`.
- `resources/scss/src/apps/agentos/fleet/cockpit/Container.scss:185-204, 309-343` — styles `.neo-button.fm-fleet-start` as the primary `--fm-signal` button and hides `.neo-button-text` below `760px` container width (`@container fm-cockpit (max-width: 760px)`).

Prescription checked: `apps/agentos/util/FleetStartPlan.mjs`, `apps/agentos/view/fleet/cockpit/ControlToolbar.mjs`, `apps/agentos/view/fleet/cockpit/FleetBatchController.mjs`, `resources/scss/src/apps/agentos/fleet/cockpit/Container.scss` — owns the concern.

## The Fix

1. **Orthogonal partition-driven visibility (`stopCount >= 1`, `startCount >= 1`):**
   - Show `■ Stop fleet · N` whenever `stop.eligible.length >= 1` (`> 0`). Hide it when `stop.eligible.length === 0`.
   - Show `▶ Start fleet · M` whenever `start.eligible.length >= 1` (`> 0`). When `start.eligible.length === 0 && stop.eligible.length > 0` (all eligible active seats are online), hide `Start fleet` so only `■ Stop fleet · N` is shown.
   - When `start.eligible.length === 0 && stop.eligible.length === 0` (roster unobserved, runtime unwired, or all seats excluded/benched), hide `Stop fleet` and show the plain `▶ Start fleet` button with its reason in `title` (preserving `#618` AC-1's fallback).
2. **Two dedicated action buttons in `ControlToolbar.mjs` and `FleetBatchController.mjs`:**
   - Place `■ Stop fleet · N` (`cls: ['fm-fleet-stop']`, `reference: 'fleet-stop-button'`, `handler: 'onStopFleet'`, `iconCls: 'fa-solid fa-stop'`, `hideMode: 'removeDom'` or `hidden: true` when `stopCount === 0`) immediately to the left of `▶ Start fleet · M` (`cls: ['fm-fleet-start']`, `reference: 'fleet-button'`, `handler: 'onStartFleet'`, `iconCls: 'fa-solid fa-play'`).
   - While `stopFleetArmed` is active, `fleet-stop-button` reads `Stop fleet · press again` (and `fleet-start-summary` shows `renderFleetStopPlan(plan)`). Pressing `Start fleet · M` disarms `stopFleetArmed` (`FleetBatchController.mjs:202`) and runs the start wave.
   - While either `startFleetPromise` or `stopFleetPromise` is in flight, both buttons keep their last settled visibility/labels and set `disabled: true`.
3. **Distinct visual hierarchy (`Container.scss`):**
   - Keep `.neo-button.fm-fleet-start` as the primary `--fm-signal` button at the right frame gutter.
   - Style `.neo-button.fm-fleet-stop` with the quiet panel base at rest (matching `.fm-reconnect-button` in `Container.scss:165-183`) and a distinct caution/danger accent on `:hover` and when armed, and include `.neo-button.fm-fleet-stop .neo-button-text` in the `@container fm-cockpit (max-width: 760px)` icon-only collapse rule (`Container.scss:325-330`) with `title` fallback.

Decision Record impact: none — consumes the existing `partitionFleetStart` / `partitionFleetStop` helpers and per-seat lifecycle intents without introducing a new Brain service or verb.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetStartPlan` (`apps/agentos/util/FleetStartPlan.mjs`) | `partitionFleetStart` & `partitionFleetStop` | Expose independent start and stop button descriptors (or `describeFleetButtons(records)` alongside `describeFleetButton`) so `startCount > 0` never masks `stopCount > 0` | `startCount === 0 && stopCount === 0` returns plain `Start fleet` with reason in `title` and hides `Stop fleet` | JSDoc | `startPlan.spec.mjs` |
| `ControlToolbar` (`apps/agentos/view/fleet/cockpit/ControlToolbar.mjs`) | Cockpit bar action group order | Render `fleet-stop-button` (`■ Stop fleet · N`) immediately left of `fleet-button` (`▶ Start fleet · M`) | Hidden when its respective partition count is `0` (except the `0/0` plain `Start fleet` fallback) | JSDoc | `fleetControl.spec.mjs` |
| `FleetBatchController` (`apps/agentos/view/fleet/cockpit/FleetBatchController.mjs`) | Settled roster + two-press stop arming | Update both buttons on settled rosters; disable both while `startFleetPromise` or `stopFleetPromise` is active; disarm stop on start press | Stale arm withdraws on roster change or 10 s timer without affecting `Start fleet · M` | JSDoc | `fleetControl.spec.mjs` |
| `.fm-fleet-stop` (`resources/scss/src/apps/agentos/fleet/cockpit/Container.scss`) | Cockpit button hierarchy & `760px` container query | Quiet panel base at rest, caution/danger accent on hover/armed; drops label to glyph below `760px` | Preserves `.fm-fleet-start` as sole `--fm-signal` primary | SCSS comments | `FleetCockpitVisual.spec.mjs` |

## Acceptance Criteria

- [ ] AC-1 — When `M >= 1` eligible seats are down and `N >= 1` eligible seats are up (e.g. 4 up, 4 down; or 1 up, 9 down), the cockpit bar renders both `Stop fleet · N` and `Start fleet · M` side by side (`startPlan.spec.mjs` + `fleetControl.spec.mjs`).
- [ ] AC-2 — When all eligible seats are down (`N === 0, M >= 1`), only `Start fleet · M` is visible; when all eligible seats are up (`N >= 1, M === 0`), only `Stop fleet · N` is visible; when `N === 0 && M === 0` (unobserved, unwired, or all excluded), `Stop fleet` is hidden and plain `Start fleet` carries the reason on `title` (`startPlan.spec.mjs` + `fleetControl.spec.mjs`).
- [ ] AC-3 — In a split fleet, pressing `Stop fleet · N` once arms `Stop fleet · press again` and writes the stop plan to `fleet-start-summary` while `Start fleet · M` remains visible; pressing `Start fleet · M` while armed disarms the stop and starts the `M` down seats; while either batch is in flight, both buttons are `disabled: true` and keep their settled labels (`fleetControl.spec.mjs`).
- [ ] AC-4 — `.fm-fleet-stop` is visually distinct from primary `.fm-fleet-start` at rest and when armed, and collapses its text label to its icon below `760px` container width (`FleetCockpitVisual.spec.mjs`).

## Out of Scope

Per-card lifecycle controls, Brain-side fleet lifecycle verbs, and automatic Claude harness update orchestration (`#682`).

## Avoided Traps

- **Keeping a single mutually exclusive button:** leaves `Stop fleet` unreachable whenever even 1 active seat is offline.
- **Styling both `Stop fleet` and `Start fleet` as twin cyan `--fm-signal` primary pills:** invites accidental fleet shutdown when reaching for `Start fleet`.
- **Hiding both buttons when `startCount === 0 && stopCount === 1`:** `stopCount >= 1` guarantees every online active seat is reachable from the bar without special-casing `N === 1`.

## Related

Parent #477 · refines #618 / PR #643 · #479 (row 2 installed walkthrough) · #682.

Live latest-open sweep: checked latest 20 open `neomjs/neo-agent-institution` issues at 2026-10-20T20:28:25Z (`#763` down to `#424`); no equivalent found.
A2A claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-20T20:28:20Z; no competing claim on the cockpit fleet buttons.
MC sweep: `query_raw_memories({query: 'neo-agent-institution Stop fleet Start fleet partial fleet both buttons ControlToolbar FleetStartPlan', nResults: 6})`, 6 results (`455a259b` / `5436edee` on `#618` single-button origin), no prior decision rejecting dual buttons.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-institution`, none overlapping.
Structure map: N/A — modifies existing files in `apps/agentos/util/`, `apps/agentos/view/fleet/cockpit/`, and `resources/scss/src/apps/agentos/fleet/cockpit/`.

unowned-rationale: filed in `FM v1` scope from operator UX challenge while `@neo-gemini-maryam` completes a queued cross-family PR review on `neomjs/neo-stage#252`; available for immediate pickup after review or by any FM cockpit builder.

Origin Session ID: 0790b2a5-994f-4fd6-a654-051d930aa37f
Retrieval Hint: "Stop fleet Start fleet split partial fleet ControlToolbar FleetBatchController FleetStartPlan describeFleetButton"

— Maryam (`@neo-gemini-maryam` · Gemini 4 Argon, Antigravity) 🧭

## Timeline

- 2026-10-20T20:30:12Z @neo-gemini-maryam added this to the **FM v1** milestone
- 2026-10-20T20:30:15Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-20T20:30:15Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-20T20:30:15Z @neo-gemini-maryam added the `ai` label
- 2026-10-20T20:30:15Z @neo-gemini-maryam added the `design` label
- 2026-10-20T20:30:16Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-20T20:30:40Z @neo-gemini-maryam added parent issue #477
- 2026-10-20T20:30:41Z @neo-gemini-maryam cross-referenced by #477
- 2026-10-20T22:39:07Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T23:02:59Z @neo-gemini-julia cross-referenced by PR #773
- 2026-10-20T23:10:29Z @neo-gemini-julia referenced in commit `a0043ff` - "style(cockpit): remove ticket archaeology refs from comments and test titles (#772)"
- 2026-10-20T23:32:46Z @neo-gemini-julia referenced in commit `8b4c84f` - "refactor(cockpit): retire single-button fleet batch shim and repoint unit tests (#772)"
- 2026-10-20T23:37:32Z @neo-gemini-julia referenced in commit `891196d` - "chore(cockpit): stamp visual baseline input hashes after RA-1 refactor (#772)"
- 2026-10-21T09:38:38Z @tobiu referenced in commit `477718f` - "feat(cockpit): show both Stop fleet and Start fleet when active fleet is split (#772) (#773)

* feat(cockpit): show both Stop fleet and Start fleet when active fleet is split (#772)

* style(cockpit): remove ticket archaeology refs from comments and test titles (#772)

* refactor(cockpit): retire single-button fleet batch shim and repoint unit tests (#772)

* chore(cockpit): stamp visual baseline input hashes after RA-1 refactor (#772)"
- 2026-10-21T09:38:39Z @tobiu closed this issue

