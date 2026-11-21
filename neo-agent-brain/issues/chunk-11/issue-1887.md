---
id: 1887
title: Batch pending normal wakes at the Antigravity idle boundary
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - performance
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-21T08:31:48Z'
updatedAt: '2026-11-21T13:24:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1887'
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
closedAt: '2026-11-21T13:24:14Z'
---
# Batch pending normal wakes at the Antigravity idle boundary

Please comment to claim before starting; check current assignees and linked PRs.

## Context

On `2026-11-21` (`08:13Z`–`08:23Z`), after `#1743` (PR `#1866`) and `#1846` (PR `#1847`) landed and bound `@neo-gemini-julia`'s Antigravity `primary-session` main chat, previously queued ordinary wake records from `2026-11-15`–`16` drained **one single record per idle turn** every ~60s (`DRAIN_RETRY_INTERVAL_MS`), even though Memory Core `unreadCount` was already `0`. Each singleton wake triggered a separate `512K`-window turn (`~275K` cached + `~10K` fresh tokens per wake) against the shared weekly Google AI Ultra pool (`17%` remaining at observation time).

`#1863` (PR `#1878`, `d820d0c4`) introduced bounded normal-wake catch-up batching (`MAX_PRIMARY_WAKE_BATCH_EVENTS = 48`, `MAX_DIGEST_BYTES = 16 KiB`) for `codex-desktop` while `#1866` was still landing for `antigravity`. Now that Antigravity `primary-session` root reconciliation and unacknowledged-turn settlement are live on `dev` (`c728772c`), Antigravity seats hit the exact singleton-spool drain that `#1863` solved for Codex.

## The Problem

Producer coalescing groups nearby arrivals before receiver acceptance, but once `WakeReceiverState` persists ordinary `wake/digest` records while an Antigravity seat is busy, restarting, or unbound across an upgrade, native delivery stages and redeems **one ordinary record per idle turn**. A returning Antigravity seat therefore spends `N` separate `CASCADE_RUN_STATUS_IDLE` turns and `PreInvocation` receipt cycles to drain `N` historical batches before catching up.

## The Architectural Reality

Checked at Brain `c728772c`:

- `ai/daemons/wake/primaryWakeBatch.mjs` (`selectPrimaryWakeBatch`, `formatPrimaryWakeBatch`), `ai/daemons/wake/primarySessionWake.mjs` (`deliverPrimarySessionWake`, `primaryWakeAttemptRecorded`), `consumeNativeWakePrompt` (`ai/daemons/wake/fleetPrimarySessionWake.mjs:223-224`), and `handleAntigravityPreInvocation` (`ai/daemons/wake/antigravitySessionWake.mjs:868-870`) are already family-agnostic over `members` and `result.digest`.
- Five `codex-desktop`-only guards exclude `antigravity` (`bareFamily` already admits `new Set(['antigravity', 'codex-desktop'])` at `ai/daemons/wake/fleetPrimarySessionWake.mjs:49`):
  1. `ai/daemons/wake/primaryWakeSpool.mjs:155` (`batchReceipt`: `binding?.family !== 'codex-desktop'`)
  2. `ai/daemons/wake/primaryWakeSpool.mjs:215` (`stagePrimaryWake`: `record.route?.adapterConfig?.primarySession?.family === 'codex-desktop'`)
  3. `ai/daemons/wake/receiver.mjs:471` (`isStagedBatchCandidate`: `currentRoute.adapterConfig?.primarySession?.family !== 'codex-desktop'`)
  4. `ai/daemons/wake/fleetPrimarySessionWake.mjs:176` (`consumeNativeWakePrompt`: `family !== 'codex-desktop'`)
  5. `ai/daemons/wake/fleetPrimarySessionWake.mjs:512` (`drainFleetPrimaryWakeRecord`: `config.family === 'codex-desktop'`)

Design authority: `#1863` (`@tobiu`, `2026-11-19`: *"batching pending normal wakes into one bounded digest when the chat goes idle"* with per-event receipts) and `#1840` Avoided Trap (*"Expiring old pending wakes to shorten the pass. It hides the cost and drops wakes a returning seat should get"*).

## The Fix

Extend `#1863`'s ordinary `primary-session` catch-up batching to `antigravity` alongside `codex-desktop` across the shared `primary-session` family set (`['codex-desktop', 'antigravity']` / shared helper) at all five gates:

1. `ai/daemons/wake/primaryWakeSpool.mjs`: allow `antigravity` in `stagePrimaryWake` candidate staging (`line 215`) and `batchReceipt` projection (`line 155`).
2. `ai/daemons/wake/receiver.mjs`: allow `antigravity` in `isStagedBatchCandidate` (`line 471`) so spool-contended staged batch candidates re-evaluate cleanly when gated.
3. `ai/daemons/wake/fleetPrimarySessionWake.mjs`: pass `commitNormalAttempt` in `drainFleetPrimaryWakeRecord` (`line 512`, invoked by `dispatchAntigravitySidecarRecords` at `line 448`) and admit `antigravity` batch members in `consumeNativeWakePrompt` (`line 176`), preserving existing `CASCADE_RUN_STATUS_IDLE`, `verifyAntigravityWakeAdmission`, `readAntigravityWakeReceipt` (`observed` and `turn-unacknowledged` with `acknowledged: false`), and urgent-slot preemption invariants.

Decision Record impact: none. Preserves the graphless Shape-B receiver boundary and signed current-main receipt contract; touches zero `AiConfig` surfaces.

### Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Antigravity ordinary `primary-session` handoff (`primaryWakeSpool.mjs`, `receiver.mjs`, `fleetPrimarySessionWake.mjs`) | `#1863` / PR `#1878` + `#1743` / PR `#1866` | Coalesce eligible waiting ordinary `wake/digest` records (up to `48` source events / `16 KiB`) into one `PreInvocation` digest when the bound Antigravity root is at `CASCADE_RUN_STATUS_IDLE`; preserve urgent preemption | Indivisible records or singletons take the existing 1-record path; overflow stays `pending` for the next idle handoff | Module/function `@summary` JSDoc in touched files | Playwright unit integration replay in `primarySessionReceiverIntegration.spec.mjs` |
| Per-event receipt settlement (`batchReceipt`, `readAntigravityWakeReceipt`) | `#1846` / PR `#1847` + `#1863` / PR `#1878` | Settle all frozen batch `members` only on verified `observed` (`private-model-ack`) or settled `turn-unacknowledged` (`acknowledged: false`) receipt; retain `inflight` until every member transitions to `delivered` | Interrupted or unproved dispatch leaves records recoverable without duplicate presentation | JSDoc in `primaryWakeSpool.mjs` / `fleetPrimarySessionWake.mjs` | Replay covering normal ack, `turn-unacknowledged`, urgent preemption, and crash/restart recovery |

## Acceptance Criteria

- [ ] Extend ordinary `primary-session` catch-up batching (`<= 48` source events, `<= 16 KiB` digest) to `antigravity` across `primaryWakeSpool.mjs`, `receiver.mjs`, and `fleetPrimarySessionWake.mjs` without introducing a duplicate family list (share a single batch-capable `primary-session` family predicate/constant).
- [ ] In `test/playwright/unit/ai/daemons/wake/primarySessionReceiverIntegration.spec.mjs` (or sibling Antigravity wake spec), replay multiple queued ordinary `wake/digest` records while an Antigravity root is `CASCADE_RUN_STATUS_RUNNING`, transition to `CASCADE_RUN_STATUS_IDLE`, and verify **one** sidecar dispatch + `PreInvocation` redemption delivers all included source events and settles each original record as `delivered` after `[NEO_WAKE_RECEIVED:<token>]`.
- [ ] Verify queue overflow (`> 48` events or `> 16 KiB`), settled `turn-unacknowledged` (`acknowledged: false` propagated across frozen members), crash/restart recovery with frozen `members`, and urgent wake preemption ahead of an unsent ordinary Antigravity batch.

## Out of Scope

- Urgent active-turn steering (`#1741`), Antigravity `2.33.0` binding regression (`#1856`), stalled busy-root telemetry (`#1854`), per-seat `appDataDir` SSOT (`#1855`), and AppleScript retirement (`#1735`).

## Avoided Traps

- **Expiring old `pending` records in `WakeReceiverState` or coupling `receiver.mjs` to Memory Core / SQLite `readAt`:** Explicitly rejected per `#1840`'s Avoided Trap (expiring pending wakes drops wakes a returning seat must receive) and `receiver.mjs`'s graphless host boundary (`receiver.mjs:7-9`). Batching drains a 48-event multi-day backlog in **1** idle turn instead of **48** without dropping any signed record.

## Related

- Builds on `#1863` (PR `#1878`), `#1743` (PR `#1866`), `#1846` (PR `#1847`), and `#1840` (PR `#1881`).
- Protects `Arm A1` `512K`-context token budget tracked on `#1853`.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: checked latest 20 open + open and closed search (`wake batch`, `stale wake`, `primaryWakeBatch`, `antigravity wake`; Out of Scope / Avoided Traps read on `#1863`, `#1840`, `#1373`) at `2026-11-21T08:30Z`; no open duplicate or exclusion found. Backlog band: `161` open (red); `#1863` is closed (`COMPLETED` for Codex only), so a dedicated Antigravity follow-up leaf is required.
MC sweep: `"stale wake PreInvocation pending receiver records unreadCount 0 primaryWakeBatch"` (10 results) + `"primary-session catch-up batch wake digest antigravity codex-desktop"` (8 results), no prior exclusion found.
Own-assignment sweep: `1` open (`#1230`, Claude Desktop `ccd-session-presets.json`), none overlapping.
A2A sweep: checked latest 30 messages across all read states (`07:29Z`–`08:29Z`); promotes `MESSAGE:2582d228-9168-4cd8-bcfc-621f8cd758cb`, no competing claim.
Structure map: `npm run ai:structure-map -- --files --loc` run; `ai/daemons/wake` owns `primaryWakeSpool.mjs` (`170` LOC), `receiver.mjs` (`941` LOC), and `fleetPrimarySessionWake.mjs` (`516` LOC); no new file.

Origin Session ID: 664ca229-4bbd-4e00-ae41-008707688181
Retrieval Hint: "Antigravity primary-session PreInvocation singleton wake spool 512K context batchReceipt selectPrimaryWakeBatch codex-desktop guard"

## Timeline

- 2026-11-21T08:31:53Z @neo-gemini-julia added the `bug` label
- 2026-11-21T08:31:53Z @neo-gemini-julia added the `good first issue` label
- 2026-11-21T08:31:53Z @neo-gemini-julia added the `help wanted` label
- 2026-11-21T08:31:53Z @neo-gemini-julia added the `ai` label
- 2026-11-21T08:31:53Z @neo-gemini-julia added the `performance` label
- 2026-11-21T08:31:54Z @neo-gemini-julia added the `model-experience` label
- 2026-11-21T08:31:54Z @neo-gemini-julia added the `agent-os` label
- 2026-11-21T08:50:40Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-21T08:54:15Z @neo-gemini-julia cross-referenced by PR #1888
- 2026-11-21T13:24:14Z @tobiu referenced in commit `d6f4310` - "feat(wake): batch normal wakes at the Antigravity idle boundary (#1887) (#1888)"
- 2026-11-21T13:24:15Z @tobiu closed this issue

