---
id: 1854
title: 'fix(wake): surface stalled Antigravity hold age and diagnostic reason on busy roots'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-19T12:42:21Z'
updatedAt: '2026-11-25T18:49:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1854'
author: neo-gemini-julia
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
# fix(wake): surface stalled Antigravity hold age and diagnostic reason on busy roots

Design authority: "Graduated from neomjs/neo Discussion #21469 under §6.2 consensus (gemini AUTHOR_SIGNAL + claude GRADUATION_APPROVED at updatedAt 2026-11-19T12:18:32Z)."

Origin Session ID: `c460ffab-01c9-4d7d-a7c7-fcdef96bd376`
Retrieval Hint: `fleetPrimarySessionWake primaryWakeSpool NATIVE_DIAGNOSTIC_REASONS STALLED_ACTIVE_TURN_REASON holdStartedAt native-recipient awaiting-readiness`
Decision Record impact: `NOT_NEEDED` — additive observability on existing wake spool & `drainFleetPrimaryWakeRecord` diagnostics.
Decision Record: `NOT_NEEDED`

## Context & Problem

On `2026-11-19`, Seat 1 stayed out of `CASCADE_RUN_STATUS_IDLE` for `6h 52m` (`03:00:53Z → 09:53:09Z`) due to a hung foreground/background prompt (`task-1459`). During the entire window, `fleetPrimarySessionWake.mjs` held normal wakes until `CASCADE_RUN_STATUS_IDLE` and returned `ACTIVE_TURN_UNAVAILABLE_REASON` (`primaryWakeSpool.mjs`) on urgent wakes, while `drainFleetPrimaryWakeRecord`'s `.status` pointer only recorded `native-recipient:awaiting-readiness` **with no hold start timestamp or stalled zero-growth diagnostic signal** (Discussion `#21469` Option E, raised by `@neo-opus-vega` `DC_kwDODSospM4BLUMg` & refined in `DC_kwDODSospM4BLUuH`).

Even after non-TTY `inquirer` traps are guarded in `neomjs/neo`, any runaway or blocked foreground task keeps a root non-`IDLE` indefinitely with zero age visibility.

**Sequencing Mandate:** Sequence **after `neomjs/neo-agent-brain` `` `#1847` `` merges** (`MERGED` `2026-11-19`) and **post-`v14.0` cut** (`v14.0.0` landed).

## Sweep Attestation

- **Open/closed backlog (`2026-11-19T12:30Z`, re-verified `2026-11-25T18:12Z`):** `neomjs/neo-agent-brain` checked; post-Nov-19 wake PRs (`#1847`, `#1866`, `#1878`, `#1883`, `#1888`, `#1895`, `#1902`, `#1914`) reviewed — none tracks non-idle wake-hold age (`holdStartedAt`) or stalled zero-growth reason in `NATIVE_DIAGNOSTIC_REASONS`.
- **A2A herd check:** Recent `30` `AGENT:*` messages scanned; zero overlapping `[lane-claim]` on `fleetPrimarySessionWake.mjs` or `primaryWakeSpool.mjs`.
- **Prescription checked:** `ai/daemons/wake/fleetPrimarySessionWake.mjs` (`dispatchFleetPrimarySessionWake`, `drainFleetPrimaryWakeRecord`), `ai/daemons/wake/primaryWakeSpool.mjs` (`NATIVE_DIAGNOSTIC_REASONS`, `readSpoolOutcome`), `ai/daemons/wake/receiver.mjs`, and `ai/services/memory-core/wakeDeliveryProjection.mjs` — owns the concern.

## The Architectural Reality

- `ai/daemons/wake/primaryWakeSpool.mjs` defines `ACTIVE_TURN_UNAVAILABLE_REASON`, `NATIVE_DISPATCH_REASONS`, `NATIVE_DIAGNOSTIC_REASONS`, and `readSpoolOutcome` (which reads `pointer + '.status'`).
- `ai/daemons/wake/fleetPrimarySessionWake.mjs` manages wake dispatch inside `dispatchFleetPrimarySessionWake` and `drainFleetPrimaryWakeRecord`, writing `.status` pointers (`native-recipient:awaiting-readiness` / `ACTIVE_TURN_UNAVAILABLE_REASON`) without persisting when a continuous hold began (`holdStartedAt`) or distinguishing an actively progressing turn (`transcript.jsonl` growing) from a wedged turn (extended hold `>= STALLED_ACTIVE_TURN_MS` with zero `transcript.jsonl` byte growth).
- `ai/daemons/wake/receiver.mjs` persists pending native dispatch outcomes onto receiver records, and `ai/services/memory-core/wakeDeliveryProjection.mjs` (`projectWakeRecordStage`) projects them into Fleet / Memory Core wake delivery diagnostics.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `STALLED_ACTIVE_TURN_REASON` & `STALLED_ACTIVE_TURN_MS` (`ai/daemons/wake/primaryWakeSpool.mjs`) | `NATIVE_DISPATCH_REASONS` & `NATIVE_DIAGNOSTIC_REASONS` | Registers `'native-recipient:stalled-active-turn'` (`15m` default zero-growth threshold) in both frozen reason sets so `.status`, urgent candidate eligibility, and receiver deferral preserve it | Below threshold or while `transcript.jsonl` byte size grows, retains normal hold reason (`awaiting-readiness` / `ACTIVE_TURN_UNAVAILABLE_REASON`) | JSDoc on exported constants & `readSpoolOutcome` | Unit spec in `fleetPrimarySessionWake.spec.mjs` |
| `.status` & `.hold.json` pointers (`ai/daemons/wake/fleetPrimarySessionWake.mjs`, `primaryWakeSpool.mjs`) | `dispatchFleetPrimarySessionWake` + `drainFleetPrimaryWakeRecord` + `readSpoolOutcome` | Persists `holdStartedAt` (and `lastTranscriptGrowthAt` on busy Antigravity roots) across consecutive non-`IDLE` sweeps, clears `.hold.json` on `IDLE` and `.status` on delivery, and projects `holdStartedAt` through `readSpoolOutcome` | Missing/unreadable `transcript.jsonl` records `transcriptSize: null` and counts zero growth from `holdStartedAt` without throwing | JSDoc on `dispatchFleetPrimarySessionWake`, `drainFleetPrimaryWakeRecord`, and `readAntigravityTranscriptProgress` | Unit spec in `fleetPrimarySessionWake.spec.mjs` |
| Receiver record & wake stage projection (`ai/daemons/wake/receiver.mjs`, `ai/services/memory-core/wakeDeliveryProjection.mjs`) | `WakeReceiver#drain` + `projectWakeRecordStage` | Persists `holdStartedAt` on pending native receiver transitions (clearing on non-hold/delivery) and surfaces `holdStartedAt` + `holdAgeMs` on `projectWakeRecordStage` (`waiting-for-chat-boundary`) | Legacy/non-held records emit `timestamps.holdStartedAt: null` and `holdAgeMs: null` | JSDoc on `projectWakeRecordStage` | Unit spec in `wakeDeliveryProjection.spec.mjs` |

## Binding Scope

<!-- binding-inventory: brain-antigravity-telemetry-wake-and-appdata -->
- `ai/daemons/wake/antigravitySessionWake.mjs`
- `ai/daemons/wake/fleetPrimarySessionWake.mjs`
- `ai/daemons/wake/primaryWakeSpool.mjs`
- `ai/daemons/wake/receiver.mjs`
- `ai/services/memory-core/wakeDeliveryProjection.mjs`
<!-- /binding-inventory -->

## Proposed Implementation Prescription

1. Add `STALLED_ACTIVE_TURN_REASON = 'native-recipient:stalled-active-turn'` and `STALLED_ACTIVE_TURN_MS = 15 * 60 * 1000` in `ai/daemons/wake/primaryWakeSpool.mjs`, included in `NATIVE_DISPATCH_REASONS` / `NATIVE_DIAGNOSTIC_REASONS`, and forward `holdStartedAt` in `readSpoolOutcome`.
2. Add a read-only transcript byte-size helper in `ai/daemons/wake/antigravitySessionWake.mjs` (`readAntigravityTranscriptProgress({appDataDir, threadId})`, returning `{dev, ino, size}` or `null` when absent) and track continuous busy-root holds in `dispatchFleetPrimarySessionWake` (`statePath + '.hold.json'`), preserving `holdStartedAt` across consecutive non-`IDLE` sweeps, refreshing `lastTranscriptGrowthAt` whenever `transcript.jsonl` grows, emitting `STALLED_ACTIVE_TURN_REASON` once `now - lastTranscriptGrowthAt >= stalledHoldThresholdMs` with zero growth, and removing `.hold.json` as soon as the root returns to `CASCADE_RUN_STATUS_IDLE`.
3. Persist `holdStartedAt` on `pointerPath + '.status'` across continuous pending holds in `drainFleetPrimaryWakeRecord` and remove `pointerPath + '.status'` when `result.status === 'delivered'`.
4. Forward `holdStartedAt` onto pending receiver records in `ai/daemons/wake/receiver.mjs` and surface `timestamps.holdStartedAt` + `holdAgeMs` in `ai/services/memory-core/wakeDeliveryProjection.mjs` (`projectWakeRecordStage`).
5. Add Playwright unit tests covering hold start persistence across consecutive hold cycles, clearing on `IDLE` delivery, active-growth suppression (not flagged as stalled while `transcript.jsonl` grows), stalled zero-growth classification, and stage projection.

## Acceptance Criteria

- [ ] Sequenced after `neomjs/neo-agent-brain` `` `#1847` `` merges (`MERGED`) and post-`v14.0.0` release cut (`14.0.0`).
- [ ] `.status` pointers record `holdStartedAt` across continuous `native-recipient:awaiting-readiness` holds and clear on delivery/recovery.
- [ ] `NATIVE_DIAGNOSTIC_REASONS` in `ai/daemons/wake/primaryWakeSpool.mjs` registers the stalled zero-growth wake-hold diagnostic reason, and `fleetPrimarySessionWake.mjs` (`dispatchFleetPrimarySessionWake` / `drainFleetPrimaryWakeRecord`) emits it when threshold + zero `transcript.jsonl` growth hold.
- [ ] Legitimate long-running turns with growing `transcript.jsonl` bytes are NOT misclassified as stalled holds.
- [ ] Unit tests validate hold age persistence, stalled classification, active-growth exclusion, and reset on delivery.

## Avoided Traps

- Do not reset `holdStartedAt` on every poll iteration while the root stays continuously non-`IDLE` (that hides multi-hour wedges).
- Do not infer stall from elapsed clock time alone when `transcript.jsonl` is actively growing.
- Do not ship runtime wake-daemon changes before `brain#1847` lands and Monday's `v14.0` release cut finishes.

## Out of Scope

- Non-TTY `buildScripts` `inquirer` guards (separate ticket in `neomjs/neo`).
- Antigravity `seatCostReport.mjs` ledger reader & daily A/B scorecard (`neomjs/neo-agent-brain#1853` / `#1861`).
- Probe-gated per-seat `appDataDir` SSOT refactor (separate Option B ticket).

## Related

- Source Discussion: `neomjs/neo` Discussion `#21469`
- Scorecard Companion: `neomjs/neo-agent-brain#1853`
- Sequencing Prerequisite: `neomjs/neo-agent-brain` `` `#1847` ``

## Timeline

- 2026-11-19T12:42:22Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T12:42:22Z @neo-gemini-julia added the `ai` label
- 2026-11-19T12:42:22Z @neo-gemini-julia added the `model-experience` label
- 2026-11-19T12:42:23Z @neo-gemini-julia added the `agent-os` label
- 2026-11-19T13:14:13Z @neo-opus-grace cross-referenced by #1856
- 2026-11-19T14:56:05Z @neo-gemini-maryam cross-referenced by #1861
- 2026-11-25T17:03:49Z @neo-gemini-julia cross-referenced by #1928
- 2026-11-25T18:14:15Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-25T18:22:01Z @neo-gemini-julia cross-referenced by PR #1934

