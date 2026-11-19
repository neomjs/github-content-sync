---
id: 1854
title: 'fix(wake): surface stalled Antigravity hold age and diagnostic reason on busy roots'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-19T12:42:21Z'
updatedAt: '2026-11-19T12:42:21Z'
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
Retrieval Hint: `fleetPrimarySessionWake primaryWakeSpool NATIVE_DIAGNOSTIC_REASONS ACTIVE_TURN_UNAVAILABLE_REASON holdStartedAt native-recipient awaiting-readiness`
Decision Record impact: `NOT_NEEDED` — additive observability on existing wake spool & `drainManifest` diagnostics.
Decision Record: `NOT_NEEDED`

## Context & Problem

On `2026-11-19`, Seat 1 stayed out of `CASCADE_RUN_STATUS_IDLE` for `6h 52m` (`03:00:53Z → 09:53:09Z`) due to a hung foreground/background prompt (`task-1459`). During the entire window, `fleetPrimarySessionWake.mjs` held normal wakes until `CASCADE_RUN_STATUS_IDLE` and returned `ACTIVE_TURN_UNAVAILABLE_REASON` (`primaryWakeSpool.mjs`) on urgent wakes, while `drainManifest`'s `.status` pointer only recorded `native-recipient:awaiting-readiness` **with no hold start timestamp or stalled zero-growth diagnostic signal** (Discussion `#21469` Option E, raised by `@neo-opus-vega` `DC_kwDODSospM4BLUMg` & refined in `DC_kwDODSospM4BLUuH`).

Even after non-TTY `inquirer` traps are guarded in `neomjs/neo`, any runaway or blocked foreground task keeps a root non-`IDLE` indefinitely with zero age visibility.

**Sequencing Mandate:** Sequence **after `neomjs/neo-agent-brain` `` `#1847` `` merges**, and **deploy post-`v14.0` cut** so live Gemini wake transport paths are untouched during release-cut work.

## Sweep Attestation

- **Open/closed backlog (`2026-11-19T12:30Z`):** `neomjs/neo-agent-brain` checked (`158` open); checked adjacent `` `#1846` `` / `` `#1847` `` — no existing issue tracks non-idle wake-hold age (`holdStartedAt`) or stalled zero-growth reason in `NATIVE_DIAGNOSTIC_REASONS`.
- **A2A herd check:** Recent `30` `AGENT:*` messages scanned; zero overlapping `[lane-claim]` on `fleetPrimarySessionWake.mjs` or `primaryWakeSpool.mjs`.
- **Code inspection:** Verified `ai/daemons/wake/primaryWakeSpool.mjs:17-25` (`ACTIVE_TURN_UNAVAILABLE_REASON`, `NATIVE_DIAGNOSTIC_REASONS`) and `ai/daemons/wake/fleetPrimarySessionWake.mjs:77-105,173,265-295,355-425` (`createPrimarySessionWakeTransport`, `drainManifest`).

## The Architectural Reality

- `ai/daemons/wake/primaryWakeSpool.mjs` defines `ACTIVE_TURN_UNAVAILABLE_REASON` and `NATIVE_DIAGNOSTIC_REASONS`.
- `ai/daemons/wake/fleetPrimarySessionWake.mjs` manages wake dispatch inside `createPrimarySessionWakeTransport` and `drainManifest`, writing `.status` pointers (`native-recipient:awaiting-readiness`) without persisting when a continuous hold began (`holdStartedAt`) or distinguishing an actively progressing turn (`transcript.jsonl` growing) from a wedged turn (extended hold with zero `transcript.jsonl` byte growth).

## Binding Scope

<!-- binding-inventory: brain-antigravity-telemetry-wake-and-appdata -->
- `ai/daemons/wake/fleetPrimarySessionWake.mjs`
- `ai/daemons/wake/primaryWakeSpool.mjs`
<!-- /binding-inventory -->

## Proposed Implementation Prescription

1. Anchor modifications by symbol across:
   - `ai/daemons/wake/fleetPrimarySessionWake.mjs` (`createPrimarySessionWakeTransport`, `drainManifest`)
   - `ai/daemons/wake/primaryWakeSpool.mjs` (`NATIVE_DIAGNOSTIC_REASONS`, `ACTIVE_TURN_UNAVAILABLE_REASON`)
2. Persist `holdStartedAt` (preserving the initial hold timestamp across consecutive retry sweeps while `awaiting-readiness` / `ACTIVE_TURN_UNAVAILABLE_REASON` persists, and clearing it once delivery succeeds or `IDLE` resumes) on `.status` pointers.
3. Add a dedicated stalled-hold diagnostic reason to `NATIVE_DIAGNOSTIC_REASONS` in `ai/daemons/wake/primaryWakeSpool.mjs` emitted when a continuous non-idle hold crosses a bounded threshold with zero `transcript.jsonl` growth (e.g., comparing stored byte offset + mtime/size vs. `holdStartedAt`).
4. Surface `holdStartedAt` / hold age in fleet wake status diagnostics without disturbing normal `IDLE` wake dispatch or triggering false alerts on legitimately long active turns that append to `transcript.jsonl`.
5. Add unit tests covering hold start persistence across consecutive hold cycles, clearing on `IDLE` delivery, active-growth suppression (not flagged as stalled while `transcript.jsonl` grows), and stalled zero-growth classification.

## Acceptance Criteria

- [ ] Sequenced after `neomjs/neo-agent-brain` `` `#1847` `` merges; deployed only post-`v14.0` cut.
- [ ] `.status` pointers record `holdStartedAt` across continuous `native-recipient:awaiting-readiness` holds and clear on delivery/recovery.
- [ ] `NATIVE_DIAGNOSTIC_REASONS` in `ai/daemons/wake/primaryWakeSpool.mjs` registers the stalled zero-growth wake-hold diagnostic reason, and `fleetPrimarySessionWake.mjs` (`createPrimarySessionWakeTransport` / `drainManifest`) emits it when threshold + zero `transcript.jsonl` growth hold.
- [ ] Legitimate long-running turns with growing `transcript.jsonl` bytes are NOT misclassified as stalled holds.
- [ ] Unit tests validate hold age persistence, stalled classification, active-growth exclusion, and reset on delivery.

## Avoided Traps

- Do not reset `holdStartedAt` on every poll iteration while the root stays continuously non-`IDLE` (that hides multi-hour wedges).
- Do not infer stall from elapsed clock time alone when `transcript.jsonl` is actively growing.
- Do not ship runtime wake-daemon changes before `brain#1847` lands and Monday's `v14.0` release cut finishes.

## Out of Scope

- Non-TTY `buildScripts` `inquirer` guards (separate ticket in `neomjs/neo`).
- Antigravity `seatCostReport.mjs` ledger reader & daily A/B scorecard (`neomjs/neo-agent-brain#1853`).
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

