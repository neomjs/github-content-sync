---
id: 1901
title: Aborted or prior-launch Antigravity wake turns wedge the seat slot
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-22T20:13:40Z'
updatedAt: '2026-11-22T20:54:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1901'
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
closedAt: '2026-11-22T20:54:47Z'
---
# Aborted or prior-launch Antigravity wake turns wedge the seat slot

## Context

On `2026-11-22`, `@neo-gemini-julia`'s native Antigravity `primary-session` wake route stopped delivering both urgent and ordinary wakes (`10` queued wakes held behind `native-recipient:receipt-unproven` and `previous launch has an unresolved wake`), while `@neo-gemini-maryam` remained reachable on the legacy `osascript` route scheduled for retirement in `#1735`.

Read-only inspection of the wedged seat state and post-cursor `transcript.jsonl` records isolated two interacting failure modes left open after `#1846` (PR `#1847`, `8f5fd0b2`):

1. On `generation A`, an ordinary wake was dispatched (`step 15643` `SYSTEM_MESSAGE`) and redeemed by `PreInvocation` (`.claim.json` recorded `admitted: true`), ran for `59` minutes across tool calls and upstream `ERROR_MESSAGE` retries, emitted `PLANNER_RESPONSE` (`tool_calls: 1`) at `step 15672` followed by its tool output `GENERIC` (`status: 'DONE'`) at `step 15673`, and then aborted to `CASCADE_RUN_STATUS_IDLE` without a final tool-less `PLANNER_RESPONSE`.
2. Nine minutes later, the Antigravity app restarted on `generation B` (whose new main chat bound cleanly in `.starts.json` with `sequence: 1, witness: true`), and an urgent wake arrived in `.urgent.request.json`.

## The Problem

1. **Non-`PLANNER_RESPONSE` idle turn endings never settle in `readAntigravityWakeReceipt`:** `#1847` (`ai/daemons/wake/antigravitySessionWake.mjs:276-281`) resets `terminalResponse = false` on *any* record other than a tool-less `PLANNER_RESPONSE`. Whenever an Antigravity wake turn aborts or stops after a tool result (`PLANNER_RESPONSE` with `tool_calls` followed by `GENERIC`), terminates on an `ERROR_MESSAGE` / `ERROR` / `CANCELED` step before or during the loop, or appends a trailing non-model bookkeeping step after a tool-less `PLANNER_RESPONSE`, `terminalResponse` stays `false` even after `conversation_summaries` returns to `CASCADE_RUN_STATUS_IDLE` — leaving `inflight` / `urgentInflight` at `{status: 'unknown'}` indefinitely.
2. **Admitted prior-launch or killed-thread Antigravity attempts never reconcile across App restarts:** When the App restarts onto a new verified lease `generation`, `dispatchFleetPrimarySessionWake` (`ai/daemons/wake/fleetPrimarySessionWake.mjs:262-274`) rightly refuses to *withdraw* (`inflight: null`) an `admitted: true` grant because its digest was already disclosed to the prior launch. However, `readReceipt` (`lines 405-417`) does not treat a dead owning `generation` (`binding.generation !== live.generation`) or a `killed === 1` summary row (where `readAntigravityWakeTarget` returns `null`) as terminal for an admitted Antigravity idle wake that never echoed `[NEO_WAKE_RECEIVED:<token>]`. If the previous launch exited mid-tool-call, before `PLANNER_RESPONSE`, or with a missing/unflushed transcript, `prior.inflight` remains `unknown` forever and every wake on the new launch fails with `previous launch has an unresolved wake`.

## The Architectural Reality

Design authority: "An `unacknowledged` receipt (the wake started a turn that ended without the hook redeeming it, or an admitted Antigravity idle wake whose finished turn never echoed its private receipt) settles the slot as delivered with `acknowledged: false`, so no later wake waits behind it." (`ai/daemons/wake/primarySessionWake.mjs:134-136`; see also `#1824` and `#1846`).

Checked at Brain `d6f43108` (`npm run ai:structure-map -- --files --loc` confirms `ai/daemons/wake/` owns `antigravitySessionWake.mjs` at `709` LOC and `fleetPrimarySessionWake.mjs` at `517` LOC):

- `ai/daemons/wake/antigravitySessionWake.mjs:119-126` (`readAntigravityWakeTarget`) returns `null` when `row.killed !== 0`, hiding `killed === 1` termination from `readReceipt` while keeping `readAntigravityWakeTarget` safe for target selection.
- `ai/daemons/wake/antigravitySessionWake.mjs:253-285` (`readAntigravityWakeReceipt`) guards two real races that MUST stay intact: (a) right after `agentapi send-message` writes `SYSTEM_MESSAGE`, SQLite `conversation_summaries.status` can briefly still read `CASCADE_RUN_STATUS_IDLE` before the cascade enters `RUNNING`; and (b) when the latest record is a `PLANNER_RESPONSE` with `tool_calls` (`antigravitySessionWake.spec.mjs:547-555`), tool execution is still pending.
- `ai/daemons/wake/fleetPrimarySessionWake.mjs:256-298, 378-422, 540-551`: `observePrimaryWakeResident` proves the live seat process `generation` before dispatch; `drainFleetPrimaryWakeRecord` already drains `normalPath` behind an active `urgentPath` when `sessionState.inflight.recordKey === normalPointer.recordKey`, so once `readReceipt` settles the dead launch's `inflight` slot as `unacknowledged` (`delivered`, `acknowledged: false`), the next pass retires the slot, binds the new `generation` root, and dispatches queued urgent and batched ordinary wakes. Codex (`isCodex`) behavior from `#1824` AC-3 remains untouched.

## The Fix

1. **`ai/daemons/wake/antigravitySessionWake.mjs`:**
   - Export `readAntigravityWakeThreadState({databasePath, projectId, threadId})` (read-only) reporting `{status, terminated: true}` when the project's top-level summary row has `killed === 1` and `{status, terminated: false}` when `eligible(row)` holds, leaving `readAntigravityWakeTarget` unchanged (`null` on `killed === 1`).
   - In `readAntigravityWakeReceipt`, replace `terminalResponse` with `turnProgressed` + `awaitingToolCalls` plus optional `ownerTerminated = false`:
     - A tool-less `PLANNER_RESPONSE` sets `turnProgressed = true, awaitingToolCalls = false` (not cleared by trailing non-`PLANNER_RESPONSE` bookkeeping records); a `PLANNER_RESPONSE` with `tool_calls` sets `turnProgressed = true, awaitingToolCalls = true`.
     - Post-wake `ERROR_MESSAGE`, `status === 'ERROR' | 'CANCELED' | 'CANCELLED'`, or a `GENERIC` tool-output record arriving after `awaitingToolCalls` sets `turnProgressed = true, awaitingToolCalls = false`.
     - When `settleUnacknowledged` is true and no `private-model-ack` was observed, return `{outcome: 'turn-unacknowledged'}` when `wakeSeen` holds and `superseded || ownerTerminated || (matched !== null && turnProgressed && !awaitingToolCalls && targetStatus === 'CASCADE_RUN_STATUS_IDLE')`.
2. **`ai/daemons/wake/fleetPrimarySessionWake.mjs`:**
   - In `dispatchFleetPrimarySessionWake`'s `readReceipt`, evaluate negative claims (`claim.admitted === false -> status: 'not-sent'`) without requiring `.cursor.json`, and for Antigravity idle attempts (`!isCodex && !activeTurn`):
     - On unredeemed attempts (`claim?.nonce !== nonce`), pass `ownerTerminated: state?.terminated === true` (never `binding.generation !== generation`, so unredeemed wakes across an App restart continue to withdraw and re-send to the new launch's root via lines `262-274`).
     - On admitted attempts (`claim.admitted === true`), first check `readAntigravityWakeReceipt` with `ownerTerminated: binding.generation !== generation || state?.terminated === true` so any echoed `[NEO_WAKE_RECEIVED:<token>]` still settles as `consumed`; otherwise settle as `status: 'unacknowledged'` (`acknowledged: false`) when `proof?.outcome === 'turn-unacknowledged'` or `ownerTerminated` holds (even if `.cursor.json` or the prior transcript was missing/truncated).

### Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `readAntigravityWakeThreadState({databasePath, projectId, threadId})` (`antigravitySessionWake.mjs`) | `antigravitySessionWake.mjs` | Reports `{status, terminated: true}` for `killed === 1` top-level project rows and `{status, terminated: false}` for eligible roots | `null` on missing, foreign-project, non-root, or malformed rows (`readAntigravityWakeTarget` stays `null` on `killed === 1`) | JSDoc `@summary` | Unit spec in `antigravitySessionWake.spec.mjs` |
| `readAntigravityWakeReceipt({... , ownerTerminated})` (`antigravitySessionWake.mjs`) | `primarySessionWake.mjs:134-136`, `#1846` | Preserves `private-model-ack`; settles `turn-unacknowledged` on finished idle turns after tool-result / error / cancel steps or `ownerTerminated` while keeping startup (`wake`-only) and pending-`tool_calls` guards `unknown` | `{outcome: 'unknown'}` while turn is running or awaiting tool output on a live owner | JSDoc `@summary` + `@param` | Unit spec in `antigravitySessionWake.spec.mjs` |
| `dispatchFleetPrimarySessionWake` (`readReceipt`, `!isCodex`) (`fleetPrimarySessionWake.mjs`) | `#1824`, `#1846`, `#1887` | Settles admitted prior-generation (`binding.generation !== generation`), `killed === 1`, or aborted idle Antigravity wakes without model echo as `delivered` (`acknowledged: false`); keeps unredeemed cross-generation withdrawal and Codex `#1824` AC-3 intact | `unknown` while current-generation live turn is still active | Inline + JSDoc `@summary` | Unit + receiver integration specs in `fleetPrimarySessionWake.spec.mjs` & `primarySessionReceiverIntegration.spec.mjs` |

Decision Record impact: aligned-with neomjs/neo-agent-brain#adr:0002

## Acceptance Criteria

- [ ] **AC-1:** `readAntigravityWakeReceipt` (`settleUnacknowledged: true`) returns `{outcome: 'turn-unacknowledged'}` when an Antigravity wake turn returns to `CASCADE_RUN_STATUS_IDLE` after `PLANNER_RESPONSE(tool_calls) -> GENERIC` tool output, `ERROR_MESSAGE` / `ERROR` / `CANCELED` steps, or trailing bookkeeping steps after a tool-less `PLANNER_RESPONSE`, while continuing to return `{outcome: 'unknown'}` on bare newly-dispatched wakes (`turnProgressed === false`) and on trailing unreturned `PLANNER_RESPONSE(tool_calls)` steps on a live owner.
- [ ] **AC-2:** Across an Antigravity App `generation` restart (`prior.binding.generation !== live.generation`) or `killed === 1` thread termination, an `admitted: true` idle wake from the dead owner that never echoed `[NEO_WAKE_RECEIVED:<token>]` settles as `delivered` with `acknowledged: false` (while preserving `consumed` when the echo *is* present, preserving withdrawal + re-delivery on unredeemed cross-generation wakes, and leaving Codex `#1824` AC-3 untouched).
- [ ] **AC-3:** End-to-end replay of the live deadlock (`prior.binding.generation` holding an `admitted: true` unacknowledged ordinary `inflight` while a newer `generation` stages both an `.urgent.request.json` and queued ordinary batch candidates) settles the prior launch's record with `acknowledged: false`, rebinds to the new `generation` root, and delivers both the urgent wake and the coalesced ordinary catch-up batch.
- [ ] **AC-4:** Unit and integration specs in `test/playwright/unit/ai/daemons/wake/antigravitySessionWake.spec.mjs`, `fleetPrimarySessionWake.spec.mjs`, and `primarySessionReceiverIntegration.spec.mjs` cover AC-1 through AC-3 and pass via `npm run test-unit`.
- [ ] **AC-5 (post-merge / installed runtime):** After host runtime refresh, `@neo-gemini-julia`'s wedged `primary-sessions/neo-gemini-julia.json` reconciles automatically to the live `generation` root and drains pending wakes without manual state-file edits.

## Out of Scope

- Urgent active-turn wake steering (`#1741`), stalled busy-root hold-age telemetry (`#1854`), per-seat `appDataDir` SSOT (`#1855`), and AppleScript transport removal (`#1735`, which this fix unblocks).

## Avoided Traps

- **Editing or deleting live `<stateDir>/primary-sessions/<seat>.json` files manually:** Violates receiver lock ownership (`#1824` Avoided Trap) and masks the recovery defect instead of proving the runtime self-heals on upgrade.
- **Settling bare `wakeSeen && targetStatus === 'CASCADE_RUN_STATUS_IDLE'` with zero post-wake progress on a live generation:** Would race the brief window between `agentapi send-message` appending `SYSTEM_MESSAGE` and the cascade transitioning to `CASCADE_RUN_STATUS_RUNNING`.
- **Withdrawing (`inflight: null`) an `admitted: true` cross-generation wake instead of settling `acknowledged: false`:** Would re-disclose an already-injected digest to the new session; settling as `delivered` (`acknowledged: false`) preserves single-disclosure while unblocking subsequent wakes.
- **Altering Codex (`isCodex`) admitted-claim settlement:** `#1824` AC-3 intentionally forbids `unacknowledged` settlement on admitted Codex claims because Codex writes hook context directly to the rollout file at turn start.

## Related

- Builds on `#1846` (PR `#1847`), `#1824` (PR `#1825`), and `#1887` (PR `#1888`).
- Unblocks `#1735` and `#759`.

Owned critical wake reliability fix requested by operator (`@tobiu`) to restore native `primary-session` wake delivery on Antigravity and unblock `#1735` (`good first issue` / `help wanted` intentionally omitted).

Live latest-open + closed sweep: checked latest 20 open + open and closed search (`antigravity wake`, `primary-session unresolved`, `antigravity wake receipt`; `Out of Scope` / `Avoided Traps` read on `#1854`, `#1855`, `#1735`, `#1846`, `#1887`, `#1824`) at `2026-11-22T20:12Z`; `161` open (red band), no open owner covers aborted Antigravity turn endings or cross-generation admitted-slot reconciliation (`#1846` is closed; `#1854` owns busy-turn hold-age telemetry only).
MC sweep: `"Antigravity wake messages wedged inflight unknown CASCADE_RUN_STATUS_IDLE previous launch unresolved wake"` (`5` results) + `"primary-session wake receipt generation unresolved unacknowledged antigravitySessionWake fleetPrimarySessionWake"` (`5` results), no prior exclusion found.
Own-assignment sweep: `1` open (`#1230` Claude Desktop folder restore), none overlapping.
A2A sweep: checked latest `30` messages across all read states (`19:37Z`–`20:11Z`); zero competing `[lane-claim]` / `[lane-intent]`.
Structure map: `npm run ai:structure-map -- --files --loc` run; `ai/daemons/wake/` owns `antigravitySessionWake.mjs` (`709` LOC) and `fleetPrimarySessionWake.mjs` (`517` LOC); zero new source files.

Origin Session ID: 12bb6bb7-21ca-4b36-824f-66ea4320e1d8
Retrieval Hint: "Antigravity primary-session wake wedged previous launch has an unresolved wake turn-unacknowledged PLANNER_RESPONSE GENERIC CASCADE_RUN_STATUS_IDLE generation restart"

## Timeline

- 2026-11-22T20:13:40Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-22T20:13:42Z @neo-gemini-julia added the `bug` label
- 2026-11-22T20:13:42Z @neo-gemini-julia added the `ai` label
- 2026-11-22T20:13:43Z @neo-gemini-julia added the `regression` label
- 2026-11-22T20:13:43Z @neo-gemini-julia added the `agent-os` label
- 2026-11-22T20:21:51Z @neo-gemini-julia cross-referenced by PR #1902
- 2026-11-22T20:54:47Z @tobiu referenced in commit `461daa3` - "fix(wake): settle aborted and prior-launch Antigravity wake turns (#1901) (#1902)"
- 2026-11-22T20:54:48Z @tobiu closed this issue

