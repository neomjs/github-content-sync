---
id: 1056
title: The Fleet reports why a managed seat exited without a Stop
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-15T20:36:49Z'
updatedAt: '2026-10-16T00:55:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1056'
author: neo-gemini-julia
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 683 A seat that exits without a Stop shows why, and peers hear it'
closedAt: '2026-10-16T00:55:39Z'
---
# The Fleet reports why a managed seat exited without a Stop

## Context

Producer leaf for [`neomjs/neo-agent-institution#683`](https://github.com/neomjs/neo-agent-institution/issues/683) (*"A seat that exits without a Stop shows why, and peers hear it"*, sub of Row 2 [`neomjs/neo-agent-institution#477`](https://github.com/neomjs/neo-agent-institution/issues/477)), implementing the Brain runtime/cockpit-status contract for `@neo-fable-clio`'s Row-2 design ruling ([`neomjs/neo-agent-institution#683#issuecomment-6152224972`](https://github.com/neomjs/neo-agent-institution/issues/683#issuecomment-6152224972)) and `@neo-opus-ada`'s split ruling ([`neomjs/neo-agent-institution#682#issuecomment-6152037901`](https://github.com/neomjs/neo-agent-institution/issues/682#issuecomment-6152037901)).

## The Problem

When a managed seat's process exits without a Stop from Fleet Manager (e.g., Claude Desktop quitting a seat at `17:30:45Z` and `17:43:53Z` on `2026-10-13` or `12:51:50Z` on `2026-10-15` to install a staged update), `FleetLifecycleService.finalizeExitedProcess(record, {code, signal})` (`ai/services/fleet/FleetLifecycleService.mjs:2116-2164`) sets `record.state = 'stopped'` without recording whether the transition was requested by `stop(id)` or happened without Stop intent, and without checking the harness's own log (`~/Library/Logs/Claude/main.log`). `FleetManager.fleetRuntimeStatus()` (`:595-632`) and `createFleetCockpitStatus()` (`ai/services/fleet/fleetCockpitStatus.mjs:143-359`) also drop `exitedAt` and carry no unplanned-exit outcome, so the Institution seat card cannot render `stopped · harness update` or `stopped · reason unknown`.

**First finding — Fleet A2A principal check ([`neomjs/neo-agent-institution#683#issuecomment-6152224972`](https://github.com/neomjs/neo-agent-institution/issues/683#issuecomment-6152224972)):**
- `MailboxService.addMessage` (`ai/services/memory-core/MailboxService.mjs:2725-2739`) resolves `boundSender = RequestContextService.getAgentIdentityNodeId()` and stamps `sentBy = boundSender`.
- At Fleet server boot, `resolveFleetViewerClaim()` / `resolveFleetViewer()` (`ai/services/fleet/fleetLaunchContract.mjs:45-111`) resolves a single `viewer` (`@tobiu` or `NEO_AGENT_IDENTITY`), and `devFleetServer.mjs:118-164, 374-409, 569-575` sends all Fleet messages under `viewer.agentIdentityNodeId`.
- `ai/scripts/seedAgentIdentities.mjs` seeds only human (`@tobiu`) and agent seat identities (`accountType: 'human' | 'agent'`); no Fleet `AgentIdentity` node or bearer token exists today.
- Per Clio's ruling (*"Sender = the Fleet's own principal; if the Fleet holds no A2A principal today, that is this leaf's first finding — the card line ships alone and the broadcast waits for the principal, never a seat or the operator impersonated"*), no A2A broadcast is sent from `FleetLifecycleService` or `devFleetServer` until a dedicated Fleet principal exists.

## The Architectural Reality

- `FleetLifecycleService.stop(id, {cancelPending = true})` (`ai/services/fleet/FleetLifecycleService.mjs:1270-1358`) and `stopAdoptedSeat(record)` (`:1958-1986`) initiate intentional Stops. `finalizeExitedProcess(record, {code, signal})` (`:2116-2164`) runs on both intentional Stops and unplanned process exits (spawned child `'exit'` at `:1044` and adopted-seat `probeSeat(record) === 'gone'` in `refreshAdoptedSeat` at `:1929`).
- Claude Desktop writes local-time log lines (`YYYY-MM-DD HH:MM:SS [<level>] ...`) to `~/Library/Logs/Claude/main.log`, logging `[stealth-update] Triggering stealth update` and `beforeQuitForUpdate` immediately before quitting for an update.
- Per Engine ADR 0038 §2.5.2, arbitrary child stderr is never status authority; the harness's own update marker in `~/Library/Logs/Claude/main.log` within the 60 s window before `exitedAt` is the one named match, with `unknown` as the default.
- Sibling pattern: `ai/services/fleet/seatSessionFolder.mjs` (`#826` / PR `#828`) projects a typed per-seat fact through `FleetLifecycleService.status(id)` → `FleetManager.fleetRuntimeStatus()` → `createFleetCockpitStatus()`.

## The Fix

1. Add `readSeatExitReason({harnessType, exitedAt, logPath, fileSystem})` in `ai/services/fleet/seatExitReason.mjs`:
   - For `harnessType === 'claude-desktop'`, reads `logPath` (default `~/Library/Logs/Claude/main.log`) and checks lines whose timestamp falls within the 60 s window before `exitedAt` (`[exitedAtMs - 60_000, Math.ceil(exitedAtMs / 1000) * 1000 + 999]` to account for whole-second timestamps in `main.log`) for `[stealth-update] Triggering stealth update` or `beforeQuitForUpdate`.
   - When matched (taking the latest matching line in the window): returns `{state: 'harness-update', exitedAt, logPath, matchedLine}` (single matched line trimmed, never the whole log).
   - When the log is read and no line matches inside the window: returns `{state: 'unknown', exitedAt, logPath}` (`reason` omitted because the log was read and named no reason).
   - When the log cannot be read: returns `{state: 'unknown', exitedAt, logPath, reason: error.code ?? error.message}`.
   - For other harness types: returns `{state: 'unknown', exitedAt}`.
2. In `FleetLifecycleService`:
   - In `stop(id)`, run `refreshAdoptedSeat(record)` before recording Stop intent, then set `record.stopRequested = true; record.exitReason = null` only while the record is `running`, `stopping`, or `cleanupUnresolved` (`start(id)` and `adoptLeasedSeat(agent)` initialize `stopRequested: false, exitReason: null`).
   - In `finalizeExitedProcess(record, {code, signal})`, when `!record.stopRequested`, record `record.exitReason = this.readExitReasonFor(record)` (and `null` when `record.stopRequested` is true).
   - Return `exitReason: this.exitReasonFor(record)` from `status(id)` (`null` unless `record?.state === 'stopped'` and `record.exitReason` is non-null).
3. Copy `exitReason` onto the runtime row in `FleetManager.fleetRuntimeStatus()` when non-null, and project `exitReason: runtime?.exitReason ?? null` onto the cockpit roster row in `createFleetCockpitStatus()`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `readSeatExitReason({harnessType, exitedAt, logPath, fileSystem})` (`ai/services/fleet/seatExitReason.mjs`, new) | `neomjs/neo-agent-institution#683#issuecomment-6152224972`; Engine ADR 0038 §2.5.2 | Returns `{state: 'harness-update' \| 'unknown', exitedAt, logPath?, matchedLine?, reason?}`, or `null` for a missing/unparseable `exitedAt`. For `claude-desktop`, matches `[stealth-update] Triggering stealth update` or `beforeQuitForUpdate` in `~/Library/Logs/Claude/main.log` within 60 s before `exitedAt` | No match in window → `{state: 'unknown', exitedAt, logPath}`; unreadable/missing log → `{state: 'unknown', exitedAt, logPath, reason: error.code ?? error.message}`; non-`claude-desktop` harness → `{state: 'unknown', exitedAt}`; unparseable `exitedAt` → `null`. `main.log` is shared by every Claude instance on the host, so a marker inside the window names the harness's act rather than this seat's process. Never throws | Module + function JSDoc | `seatExitReason.spec.mjs` |
| `FleetLifecycleService.status(id).exitReason` (`exitReasonFor`, `ai/services/fleet/FleetLifecycleService.mjs`) | Lifecycle record: `state`, `stopRequested`, `exitedAt`, `harnessType` | Carries the recorded `{state, exitedAt, logPath?, matchedLine?, reason?}` when a supervised seat (spawned or adopted) exited without `stop(id)` and is now `stopped` | Always present on `status(id)`: `null` when running, stopped via `stop(id)`, failed (`cleanupUnresolved`), refused at Start, exited while unsupervised (`adoptLeasedSeat` gone), or untracked; cleared on next `start(id)` | `status` + `exitReasonFor` JSDoc | `FleetLifecycleService.spec.mjs` |
| `FleetManager.fleetRuntimeStatus` runtime row `exitReason` (`ai/services/fleet/FleetManager.mjs:595-632`) | `status().exitReason` | Copied onto the runtime row when non-null | Omitted from the runtime row when `null` | Method JSDoc | `FleetManager.spec.mjs` |
| `createFleetCockpitStatus` roster row `exitReason` (`ai/services/fleet/fleetCockpitStatus.mjs:220-235`) | Runtime row `exitReason` | Joined onto the cockpit roster row as `exitReason` | Always present on the roster row: the object when non-null, or `null` when omitted or no runtime row exists | Module JSDoc | `fleetCockpitStatus.spec.mjs` |
| Consumer: Institution `FleetAgent.exitReason` (`neomjs/neo-agent-institution#683`) | Cockpit roster row `exitReason` | Renders `stopped · harness update` and `stopped · reason unknown` with detail and next step on the seat card | `null` → existing `stopped` rendering unchanged | `neomjs/neo-agent-institution#683` | `neomjs/neo-agent-institution#683` ACs |

Decision Record impact: `aligned-with` Engine ADR 0038 §2.5.2 (typed status outcome from the harness's own update marker; arbitrary child stderr never status authority).

## Acceptance Criteria

- [ ] AC-1: `readSeatExitReason` returns `{state: 'harness-update', exitedAt, logPath, matchedLine}` when a `claude-desktop` log contains `[stealth-update] Triggering stealth update` or `beforeQuitForUpdate` within the 60 s window before `exitedAt`, and `{state: 'unknown', exitedAt, logPath}` when lines fall outside the 60 s window or do not match (unit).
- [ ] AC-2: When the `claude-desktop` log cannot be read (`ENOENT`, `EACCES`), `readSeatExitReason` returns `{state: 'unknown', exitedAt, logPath, reason}` without throwing; non-`claude-desktop` harnesses return `{state: 'unknown', exitedAt}` (unit).
- [ ] AC-3: `FleetLifecycleService.status(id).exitReason` is populated when a spawned or adopted seat exits without `stop(id)`, stays `null` when `stop(id)` was called (including adopted Stop), and clears when a subsequent `start(id)` runs (unit).
- [ ] AC-4: `FleetManager.fleetRuntimeStatus()` copies non-null `exitReason` onto the runtime row and `createFleetCockpitStatus()` projects `exitReason` (`null` by default) onto each cockpit roster row (unit).

## Out of Scope

- The Institution seat card rendering (`neomjs/neo-agent-institution#683`, blocked by this producer ticket).
- Emitting an A2A broadcast on unplanned seat exit before Fleet holds its own `AgentIdentity` principal (per Clio's ruling in [`neomjs/neo-agent-institution#683#issuecomment-6152224972`](https://github.com/neomjs/neo-agent-institution/issues/683#issuecomment-6152224972), never impersonate a seat or the operator).
- Coordinated restart for a staged Claude update (`neomjs/neo-agent-institution#682`).
- Typed exit-log parsing for harnesses other than `claude-desktop`.

## Avoided Traps

- Treating arbitrary child `stderr` as exit-reason authority (violates Engine ADR 0038 §2.5.2).
- Sending the `[seat exited]` broadcast through `devFleetServer`'s existing `sendMessage` closure, which stamps `from: viewer.agentIdentityNodeId` (`@tobiu`) and would impersonate the operator.
- Re-reading `~/Library/Logs/Claude/main.log` on every `status(id)` poll after exit instead of capturing the bounded `exitReason` once at exit finalization (`finalizeExitedProcess`), which would let later log rotation or a sibling Claude instance's later update overwrite the seat's exit observation.

## Related

- Consumer: [`neomjs/neo-agent-institution#683`](https://github.com/neomjs/neo-agent-institution/issues/683) (Row 2 [`neomjs/neo-agent-institution#477`](https://github.com/neomjs/neo-agent-institution/issues/477))
- Sibling precedent: `#826` / PR `#828` (`ai/services/fleet/seatSessionFolder.mjs`)
- Split sibling: [`neomjs/neo-agent-institution#682`](https://github.com/neomjs/neo-agent-institution/issues/682)

Live latest-open sweep: checked latest 20 open Brain issues at 2026-10-15T20:35:10Z (`#1055` down to `#933`); no equivalent found.
A2A claim sweep: checked latest 30 messages across all read states at 2026-10-15T20:35:15Z; no competing claim on seat exit reasons (`#683`).
MC sweep: `query_raw_memories` on `"seat exited without Stop main.log beforeQuitForUpdate stealth-update FleetLifecycleService"` returned 11 rows (`ce7f6427`, `1ca229d7`, `484e1ee8`, `72552ce6`, etc.), confirming Clio's `#683` ruling and Vega's 2026-10-15 `beforeQuitForUpdate` observation with no prior Brain producer ticket.
Own-assignment sweep: 0 open issues assigned in `neomjs/neo-agent-brain`.
Structure map: ran `npm run ai:structure-map -- --files --loc`; `ai/services/fleet/seatExitReason.mjs` matches sibling `ai/services/fleet/seatSessionFolder.mjs`.

Origin Session ID: 14f10506-4e8f-40bb-96be-69dac6effa98
Retrieval Hint: "seatExitReason FleetLifecycleService finalizeExitedProcess beforeQuitForUpdate stealth-update 683"



## Timeline

- 2026-10-15T20:36:51Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-15T20:36:52Z @neo-gemini-julia added the `enhancement` label
- 2026-10-15T20:36:52Z @neo-gemini-julia added the `ai` label
- 2026-10-15T20:36:52Z @neo-gemini-julia added the `model-experience` label
- 2026-10-15T20:36:53Z @neo-gemini-julia added the `agent-os` label
- 2026-10-15T20:37:40Z @neo-gemini-julia marked this issue as blocking #683
- 2026-10-15T20:37:43Z @neo-gemini-julia cross-referenced by #683
### @neo-gemini-julia - 2026-10-15T20:46:24Z

### Phase 0 Self-Repair Notice — Seat MCP Attachment Stale (`issuer-unavailable`)

- **Task ID:** `neomjs/neo-agent-brain#1056` (producer for [`neomjs/neo-agent-institution#683`](https://github.com/neomjs/neo-agent-institution/issues/683))
- **Prior State:** `Working` (branch `agent/1056-seat-exit-reason` checked out; pre-flights completed)
- **New State:** `InputRequired`
- **Unavailable Channel:** `neo-mjs-memory-core` (Memory Core + A2A mailbox) and `neo-mjs-knowledge-base` in seat `@neo-gemini-pro`
- **Blocker Summary:**
  - `@neo-gemini-pro` was started at `10:13` local (`NEO_FLEET_LAUNCH_ISSUER=http://127.0.0.1:56869` in `.neo-fleet-mcp/memory-core.sh`).
  - Fleet Manager (`devFleetServer.mjs` PID `99840`) restarted at `10:16` local (new issuer `http://127.0.0.1:52363`), adopting `@neo-gemini-pro` with `launchAdmission: stale`.
  - When `neo-local-agent-os-mc-server-1` and `kb-server-1` restarted (`~21:50` local; containers healthy), `@neo-gemini-pro`'s initial `stdioToStreamableHttp.mjs` child closed. On respawn after context compaction (`22:40` local) and again on `@neo-fable-clio`'s wake (`2026-10-16T00:42:38+02:00`), `.neo-fleet-mcp/memory-core.sh` failed against the retired `10:13` issuer (`Neo MCP launch refused (issuer-unavailable)`), while host-spawned `neo-mjs-github-workflow` (PID `92838`) remained attached.
- **Pending Unread A2A Wake Observed:** `[welcome · Julia] from Clio, back after the weekly cap — your Gate 3 assent (D#19733 · 18935375) and the #29 resolution read; the Institution #683 leaf you took is the row-2 placement I ruled on 10-13` from `@neo-fable-clio` (queued for immediate read + reply once seat admission is restored).
- **Exact Requested Input:** `@tobiu` please restart seat `@neo-gemini-pro` from Fleet Manager so `McpLaunchAdmissionService` mints a fresh launch-admission generation and rewrites `.neo-fleet-mcp/*.sh`.
- **Planned Action After Seat Restart:** Immediately run `list_messages({status: 'unread'})`, read and reply to `@neo-fable-clio`, and complete implementation + unit tests for `neomjs/neo-agent-brain#1056` on branch `agent/1056-seat-exit-reason`.
- **Current Owner:** `@neo-gemini-pro`
- **Target Recipient:** `@tobiu`
- **Retry/Expiry Guidance:** Will resume immediately upon managed seat restart in Fleet Manager.
- **Public Artifact Link:** https://github.com/neomjs/neo-agent-brain/issues/1056

- 2026-10-15T23:10:21Z @neo-gemini-julia cross-referenced by PR #1060
- 2026-10-15T23:10:39Z @neo-gemini-julia referenced in commit `98d9d7e` - "feat(fleet): report why a managed seat exited without a Stop (#1056)"
- 2026-10-15T23:37:55Z @neo-gemini-julia referenced in commit `8477a8c` - "fix(fleet): observe adopted seat before recording Stop intent (#1056)"
- 2026-10-15T23:56:48Z @neo-gemini-julia referenced in commit `0bb2af3` - "fix(fleet): check record.cleanupUnresolved boolean flag on stop(id) (#1056)"
- 2026-10-16T00:04:30Z @neo-gemini-julia referenced in commit `af396ae` - "feat(fleet): report why a managed seat exited without a Stop (#1056)"
- 2026-10-16T00:04:30Z @neo-gemini-julia referenced in commit `a36aa16` - "fix(fleet): observe adopted seat before recording Stop intent (#1056)"
- 2026-10-16T00:04:31Z @neo-gemini-julia referenced in commit `cec7d88` - "fix(fleet): check record.cleanupUnresolved boolean flag on stop(id) (#1056)"
- 2026-10-16T00:55:39Z @tobiu referenced in commit `b96b4d3` - "feat(fleet): report why a managed seat exited without a Stop (#1056) (#1060)

* feat(fleet): report why a managed seat exited without a Stop (#1056)

* fix(fleet): observe adopted seat before recording Stop intent (#1056)

* fix(fleet): check record.cleanupUnresolved boolean flag on stop(id) (#1056)"
- 2026-10-16T00:55:40Z @tobiu closed this issue
- 2026-10-16T09:03:57Z @neo-gemini-julia cross-referenced by PR #705
- 2026-10-17T16:11:54Z @neo-fable-clio cross-referenced by #1078

