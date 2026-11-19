---
id: 1846
title: Antigravity wake receipts drop on internal/4KB steps and wedge slot
state: OPEN
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-11-18T22:16:25Z'
updatedAt: '2026-11-19T02:58:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1846'
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
# Antigravity wake receipts drop on internal/4KB steps and wedge slot

## Context

Operator reported on 2026-11-18 that `@neo-gemini-julia` (Antigravity) received zero confirmed wakes in Fleet Manager (`wake.state: "unknown"`), while `@neo-gemini-maryam` (Antigravity) continued showing `wake.state: "reachable"`.

Inspection of `@neo-gemini-julia`'s sessions `613064f8-74aa-4baa-8ee7-6a0b8540f31b` (step `212` at `2026-11-18T13:34:14Z`, marker `[NEO_WAKE_V1:7e0fd5b8-c94e-4dcf-9d4c-323ebf05e096] [WAKE]`) and `c460ffab-01c9-4d7d-a7c7-fcdef96bd376` (step `490` at `2026-11-18T21:25:03Z`, marker `[NEO_WAKE_V1:281b4f53-e9c2-428f-9cc4-99f2c9995db8] [WAKE]`) proves that `antigravityWakeSidecar.mjs` **did** bind both roots and dispatched one `agentapi send-message` wake per session, yet neither attempt recorded a receipt in `readAntigravityWakeReceipt`, leaving `inflight` permanently stuck at `{status: 'unknown'}` (`another delivery unresolved` for later wakes).

## The Problem

Four defects interact to drop Antigravity wake receipts on `@neo-gemini-julia` while passing on `@neo-gemini-maryam`:

1. **Compact `transcript.jsonl` `truncated_fields` resets correlation on >4 KB steps:** Every step >~4 KB in compact `transcript.jsonl` sets `truncated_fields` (e.g. `['content']`, `['thinking']`, `['tool_calls']`). Because `@neo-gemini-julia` carried `2,702` unread broadcasts, `list_messages({status: 'unread', projection: 'counts'})` at step `492` wrote `4,120` bytes (`truncated_fields: ['content']`), immediately wiping `matched = null` in `readAntigravityWakeReceipt` and `latest = false` in `hasAntigravityPendingSystemWake`. `@neo-gemini-maryam` drained broadcasts to `0` (`< 4 KB`, no `truncated_fields`).
2. **Internal mid-turn `SYSTEM_MESSAGE` steps treated as foreign turn boundaries:** `@neo-gemini-julia` runs with Plan Mode ON (`<plan_execute_protocol>`), which emits `SYSTEM_MESSAGE` records on `[plan review due]`, `[plan review]`, and `Plan mode is off` (plus async `run_command` task completions `sender=<convId>/task-*` and subagent notifications). Both `hasAntigravityPendingSystemWake` and `readAntigravityWakeReceipt` reset on **any** `item.type === 'SYSTEM_MESSAGE'`. `@neo-gemini-maryam` runs with Plan Mode OFF.
3. **Deprecated `ephemeralMessage` injection + `protojson` zero-omission:** `handleAntigravityPreInvocation` injects `{ephemeralMessage: ...}`, which is not persisted across Plan Mode model handoffs (`enter_plan_mode` / `exit_plan_mode`) and mid-turn compactions (`CHECKPOINT`), so the final `PLANNER_RESPONSE` never sees `[NEO_WAKE_RECEIVED:<token>]`. Additionally, `protojson` omits zero-valued `invocationNum` (`undefined`).
4. **Codex-only unacknowledged slot settlement (`#1825`):** `fleetPrimarySessionWake.mjs` only settles finished unacknowledged idle wake turns (`status: 'unacknowledged'`, `acknowledged: false`) when `isCodex` is true, so a single unacknowledged Antigravity wake permanently blocks all subsequent wakes in that session.

## The Architectural Reality

Design authority: `#1706` ("Require a matching private model receipt before settling delivery") and `#1824` ("No record makes ordinary delivery wait without bound behind an attempt nobody can redeem"). Placement: `ai/daemons/wake/` (confirmed via `npm run ai:structure-map -- --files --loc`).

- `ai/daemons/wake/antigravitySessionWake.mjs:197-236`: `hasAntigravityPendingSystemWake` and `readAntigravityWakeReceipt` bail/reset whenever `item.truncated_fields` is truthy or `item.type === 'SYSTEM_MESSAGE'`.
- `ai/daemons/wake/antigravitySessionWake.mjs:320-368` and `ai/daemons/wake/antigravityWakeHook.mjs:59`: `invocationNum` rejects `undefined` under `< 0` checks; `handleAntigravityPreInvocation` emits `ephemeralMessage` rather than `{systemMessage: {systemMessage: ...}}`.
- `ai/daemons/wake/fleetPrimarySessionWake.mjs:310-339`: `readReceipt` returns `null` forever when `!isCodex` and a wake turn completes without a private model acknowledgment.

## The Fix

1. **External system-envelope fencing (`ai/daemons/wake/antigravitySessionWake.mjs`):** Add `isAntigravityExternalSystemMessage(item)` recognizing only `agentapi send-message` external envelopes (`[Message] timestamp=... sender=system priority=... content=...`). Fence `hasAntigravityPendingSystemWake` and `readAntigravityWakeReceipt` only on `!item` (corrupt/oversized/skipped partial JSONL lines), `USER_INPUT`, or `isAntigravityExternalSystemMessage(item)` — ignoring `truncated_fields` on `GENERIC`/checkpoint/unrelated records and ignoring internal `SYSTEM_MESSAGE` records (Plan Mode, task/subagent notices). Guard `item.content` / `item.tool_calls` reads against matching `truncated_fields` entries (`content` / `tool_calls`).
2. **Persist wake digest + normalize `invocationNum` (`ai/daemons/wake/antigravitySessionWake.mjs`, `ai/daemons/wake/antigravityWakeHook.mjs`):** Emit `{systemMessage: {systemMessage: ...}}` from `handleAntigravityPreInvocation` so the digest and `[NEO_WAKE_RECEIVED:<token>]` survive Plan Mode handoffs and checkpoints for `private-model-ack` verification, and treat omitted `invocationNum` (`undefined`) as `0`.
3. **Antigravity unacknowledged turn settlement (`ai/daemons/wake/antigravitySessionWake.mjs`, `ai/daemons/wake/fleetPrimarySessionWake.mjs`):** When `settleUnacknowledged` is enabled and no `[NEO_WAKE_RECEIVED:...]` model ack was observed, `readAntigravityWakeReceipt` returns `{outcome: 'turn-unacknowledged'}` if the wake was superseded by a later `USER_INPUT` / external `SYSTEM_MESSAGE` or ended in a tool-less `PLANNER_RESPONSE` with `targetStatus === 'CASCADE_RUN_STATUS_IDLE'`. Wire `!isCodex` idle receipt reading in `dispatchFleetPrimarySessionWake` to settle finished unredeemed or unechoed Antigravity turns as `status: 'unacknowledged'` (`acknowledged: false`) rather than wedging the seat, keeping `#1825` Codex `isCodex` behavior unchanged.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `isAntigravityExternalSystemMessage(item)` | Antigravity native envelope | True only for `SYSTEM_MESSAGE` (`source: 'SYSTEM'`, `status: 'DONE'`, `sender=system`, ISO timestamp) | `false` on internal Plan Mode / task / subagent messages | JSDoc `@summary` | Unit spec in `antigravitySessionWake.spec.mjs` |
| `hasAntigravityPendingSystemWake(...)` | `antigravitySessionWake.mjs` | Preserves match across `truncated_fields` and internal `SYSTEM_MESSAGE`s; resets on `USER_INPUT` or external `SYSTEM_MESSAGE` | `false` on corrupt record or file error | JSDoc `@summary` | Unit spec in `antigravitySessionWake.spec.mjs` |
| `readAntigravityWakeReceipt(...)` | `#1706` / `#1824` | Matches `private-model-ack` across `truncated_fields`/internal `SYSTEM_MESSAGE`s; returns `turn-unacknowledged` on idle finished/superseded turn when `settleUnacknowledged: true` | `{outcome: 'unknown'}` while turn running or tool call active | JSDoc `@summary` | Unit specs in `antigravitySessionWake.spec.mjs` |
| `handleAntigravityPreInvocation` / `observeAntigravityPreInvocation` / `antigravityWakeHook` | `#1706` & hooks contract | Accepts `invocationNum === undefined` as `0`; injects `{systemMessage: {systemMessage: ...}}` | Refused (`deny`, terminate) on wake markers with non-integer/negative `invocationNum` or unauthorized marker; `{}` on input that is not a wake marker | JSDoc `@summary` | Unit specs in `antigravitySessionWake.spec.mjs` |
| `dispatchFleetPrimarySessionWake` (`readReceipt`, `!isCodex`) | `#1824` | Settles finished unredeemed or unechoed idle Antigravity turns as `status: 'unacknowledged'` (`acknowledged: false`) | `null` while turn running or active-turn mode | JSDoc `@summary` | Unit spec in `fleetPrimarySessionWake.spec.mjs` |

Decision Record impact: aligned-with neomjs/neo-agent-brain#adr:0002

## Acceptance Criteria

- [ ] **AC-1:** `hasAntigravityPendingSystemWake` and `readAntigravityWakeReceipt` preserve wake correlation across `truncated_fields` on `GENERIC`/non-matching records and internal `SYSTEM_MESSAGE`s (Plan Mode, task completions), while still fencing on `USER_INPUT` and external `sender=system` envelopes.
- [ ] **AC-2:** `handleAntigravityPreInvocation` injects `{systemMessage: {systemMessage: ...}}` (persisting across Plan Mode/checkpoints), accepts `protojson`-omitted `invocationNum` (`undefined -> 0`), and `readAntigravityWakeReceipt` confirms `private-model-ack` across those transitions.
- [ ] **AC-3:** `readAntigravityWakeReceipt` (`settleUnacknowledged: true`) and `dispatchFleetPrimarySessionWake` settle finished unredeemed or unechoed Antigravity idle wake turns as `delivered` with `acknowledged: false`, unblocking subsequent wakes without wedging `inflight`/`urgentInflight`, while leaving active/running turns untouched.
- [ ] **AC-4:** Unit specs in `test/playwright/unit/ai/daemons/wake/antigravitySessionWake.spec.mjs` and `fleetPrimarySessionWake.spec.mjs` cover AC-1 through AC-3 and pass under `npm run test-unit`.
- [ ] **AC-5 (post-merge / installed runtime):** After release/rebuild of the installed host runtime, an idle Antigravity seat running Plan Mode with >4 KB tool steps receives ordinary and urgent wakes cleanly (`wake.state: "reachable"`).

## Out of Scope

- `#1743` pre-start project census vs `SessionStart` ordering race.
- `#1741` active-turn wake steering.

## Avoided Traps

- Do not reset correlation on `CHECKPOINT` or `GENERIC` `truncated_fields` (only `!item` JSONL parse/skip resets `matched`, never returning early before subsequent wakes in the same stream).
- Do not withdraw/re-send an unacknowledged Antigravity wake into the same broken session (per `#1824`: settle as `unacknowledged` / `acknowledged: false` so subsequent wakes proceed).

## Related

- `#1706` (`e4bc5491`), `#1824` (`92f0e736`), `#1743`, `#1840`

Origin Session ID: c460ffab-01c9-4d7d-a7c7-fcdef96bd376
Retrieval Hint: "antigravity wake receipt truncated_fields internal SYSTEM_MESSAGE unacknowledged"

Sweeps:
- Live latest-open + closed sweep: checked latest 20 open created-descending + open/closed search (`antigravity wake OR primarySessionWake OR antigravitySessionWake OR unacknowledged`; 155 open total, red band; read `#1743` and `#1824` Out of Scope/Avoided Traps) at `2026-11-18T22:15:36Z`; `#1743` excludes post-dispatch receipt handling and `#1824` is closed and scoped to Codex only.
- A2A sweep: checked latest 25 messages across all read-states at `2026-11-18T22:15:35Z`; zero overlapping `[lane-claim]`/`[lane-intent]`.
- MC sweep: `query_raw_memories` ("wake state unknown Antigravity session did not get wake regression", 5 hits); no active duplicate fix in flight.
- Own-assignment sweep: 1 open (`#1230` Claude Desktop session restore), non-overlapping.
- Structure-map sweep: `npm run ai:structure-map -- --files --loc` identifies `ai/daemons/wake/` as the owning module location.

## Timeline

- 2026-11-18T22:16:26Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-18T22:16:27Z @neo-gemini-julia added the `bug` label
- 2026-11-18T22:16:27Z @neo-gemini-julia added the `ai` label
- 2026-11-18T22:16:28Z @neo-gemini-julia added the `regression` label
- 2026-11-18T22:16:28Z @neo-gemini-julia added the `agent-os` label
- 2026-11-18T22:24:15Z @neo-gemini-julia cross-referenced by PR #1847
- 2026-11-19T02:58:24Z @neo-gemini-julia referenced in commit `c33d20f` - "fix(wake): clarify unacknowledged settlement docs and JSDoc params (#1846)"

