---
id: 1215
title: Reconnect Neural Link after Bridge restart with fresh parent token
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T19:27:37Z'
updatedAt: '2026-10-20T10:22:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1215'
author: neo-gemini-julia
commentsCount: 1
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-20T10:22:15Z'
---
# Reconnect Neural Link after Bridge restart with fresh parent token

## Context

Promoted from `@neo-opus-vega`'s `defect-note` (`MESSAGE:59608605-51a9-4fae-ad64-a87bfaca76c6`, `2026-10-19T18:59:24Z`) and `[infra · now]` warning (`MESSAGE:cd7c70d6-504c-4163-9259-0db08179eaf7`, `2026-10-19T19:11:35Z`) under [`neomjs/neo-agent-institution#732`](https://github.com/neomjs/neo-agent-institution/issues/732) (*An FM update never requires closing a harness*):

> `defect-note: Neural Link client is wrong: after a Bridge restart no running seat gets NL back. Two causes stack:`
> `1. ConnectionService never reconnects after a close. Its close handler ends in the comment "Auto-reconnect logic could go here".`
> `2. manage_connection start is refused with 1008 once the seat's launch-minted Bridge token is past its 1 h TTL. bridgeTokenTtlMs is 3600000, and the keyed Bridge verifies expiresAt only at connect.`
> `Measured at 19:00Z: all 16 running NL children are more than 1 h old. Mine launched at 14:55:06Z, and its token expired at 15:55:06Z. So #1212's supervised mid-session restart revives a Bridge that no running seat can rejoin. Today only a seat restart restores NL. A fix needs the reconnect plus a fresh token: fleetMcpLauncher stays alive as the child's parent and holds the admission, while the child holds none.`

Design authority: [`neomjs/neo-agent-institution#732`](https://github.com/neomjs/neo-agent-institution/issues/732) (terminal predicate: an FM update or supervised Bridge recovery never requires closing or restarting a seat harness) and [`neomjs/neo-agent-brain#1203`](https://github.com/neomjs/neo-agent-brain/issues/1203) / [`#1212`](https://github.com/neomjs/neo-agent-brain/pull/1212) (`superviseFleetBridge` mid-session Bridge restart).

## The Problem

Three failure modes stack when the Neural Link Bridge restarts (either via `#1212`'s `superviseFleetBridge`, `settleFleetBridge` replacement on generation/key mismatch, or a plane/FM cut):

1. **No Auto-Reconnect & Stale Session Cache on Disconnect (`ai/services/neural-link/ConnectionService.mjs:456–469`):**
   - When the active Bridge WebSocket closes, `ws.on('close')` sets `this.bridgeSocket = null`, settles in-flight requests, and ends with `// Optional: Auto-reconnect logic could go here` (`:468`). Nothing schedules a reconnect, and `this.sessionData` (`:116`) / `this.activeAgents` (`:122`) retain stale entries from the closed Bridge connection.
2. **Static 1-Hour `NEO_FLEET_BRIDGE_TOKEN` in the Long-Lived `neural-link` Child (`ai/mcp/client/fleetMcpLauncher.mjs:115–199, 211–238` & `ai/services/neural-link/ConnectionService.mjs:316–327`):**
   - All desktop seats (`claude-desktop`, `antigravity`, `codex-desktop` — all 16 running NL children measured on the host) spawn `ai/mcp/server/neural-link/mcp-server.mjs` via `fleetMcpLauncher.mjs --server neural-link`.
   - `fleetMcpLauncher` redeems `NEO_FLEET_LAUNCH_GRANT` once at spawn, injects a 1-hour `NEO_FLEET_BRIDGE_TOKEN` (`bridgeTokenTtlMs = 3600000` in `FleetRegistryService.mjs:372`), and strips `NEO_FLEET_LAUNCH_GRANT` and `NEO_FLEET_LAUNCH_ISSUER` from the child's `process.env` (`composeTargetEnv`, `fleetMcpLauncher.mjs:117–134`).
   - Meanwhile, `McpLaunchAdmissionService#redeem` (`McpLaunchAdmissionService.mjs:55–56, 879–963`) and `createBridgeTokenOwner` (`launchCredentialOwners.mjs:151–166`) already support repeated redemptions for the lifetime of an active generation, minting a fresh 1-hour token (`registry.renewBridgeToken(binding.agentId, binding.authority).token`) on every redemption. Because `fleetMcpLauncher` stays alive as the child's direct parent (`runTarget`, `:211–238`) holding the grant while the child holds none, the child currently has no way to ask its parent for a freshly redeemed token when reconnecting after 1 hour.
3. **`ensureBridgeAndConnect` Spawns an Unkeyed `Bridge.mjs` on Auth Rejection (`ai/services/neural-link/ConnectionService.mjs:555–583`):**
   - When `connectToBridge()` is rejected with `1008 Authentication required` (e.g., an expired token on `manage_connection start`) or fails when the seat holds a Fleet Bridge token (`NEO_FLEET_BRIDGE_TOKEN`), `ensureBridgeAndConnect()`'s `catch (e)` treats any non-`protocolFreshnessFailed` error as `"Bridge not running"` and calls `this.spawnBridgeProcess()` (`:565`) — spawning an unkeyed `Bridge.mjs` without `NEO_FLEET_BRIDGE_PUBLIC_KEY` or `neural-link-bridge-owner.json`!

## The Architectural Reality

- `ai/mcp/client/fleetMcpLauncher.mjs` (`307` LOC):
  - `admitLaunch` (`:136–199`) redeems the seat's grant from `McpLaunchAdmissionService` (`POST /fleet/mcp-launch/v1`) and returns `{args, env, admittedEnv, generationRoot}`.
  - `runTarget` (`:201–228`) spawns the admitted MCP server child (`stdio: 'inherit'`) and waits for `child.once('exit')`. By opening a 4th IPC file descriptor (`stdio: ['inherit', 'inherit', 'inherit', 'ipc']`) when a `refreshEnv` callback is supplied, `runTarget` can answer `{type: 'neo-fleet-mcp-refresh-env', id}` messages from its direct child by re-running `admitLaunch` in the parent process and replying `{type: 'neo-fleet-mcp-refresh-env-result', id, ok: true, env: admittedEnv}` — never exposing `NEO_FLEET_LAUNCH_GRANT` or `NEO_FLEET_LAUNCH_ISSUER` to the child (`PROCESS_CONTROL` in `mcpLaunchAdmission.mjs:62`).
- `ai/services/neural-link/ConnectionService.mjs` (`1149` LOC):
  - `createBridgeUrl()` (`:316–327`) constructs `ws://127.0.0.1:${this.port}?role=agent&id=${this.agentId}&token=...`.
  - `connectToBridge()` (`:393–471`) manages the WebSocket handshake and `ws.on('close')`.
  - `ensureBridgeAndConnect()` (`:555–583`) connects to the Bridge and only spawns a standalone Bridge when no Fleet Bridge token is in use and the failure is not an authentication rejection (`1008`).
  - `manageConnection({action})` (`:924–954`) handles explicit `start` / `stop`.

## The Fix

1. **Parent-Held Credential Refresh over IPC (`ai/mcp/client/fleetMcpLauncher.mjs`):**
   - Have `admitLaunch` also expose `admittedEnv: payload.env` (the admitted server-owned + resident env keys without base `env`).
   - In `runTarget({args, env, execPath, spawnFn, refreshEnv})`: when `typeof refreshEnv === 'function'`, spawn the child with `stdio: ['inherit', 'inherit', 'inherit', 'ipc']` and listen for `child.on('message', msg)`:
     - On `{type: 'neo-fleet-mcp-refresh-env', id}`, call `await refreshEnv()` and `child.send({type: 'neo-fleet-mcp-refresh-env-result', id, ok: true, env: refreshed.admittedEnv})` (or `{type: 'neo-fleet-mcp-refresh-env-result', id, ok: false, code: error?.code ?? null, reason: error?.reason ?? null}` on failure), coalescing concurrent refresh requests in the parent.
   - Export a helper `requestParentEnvRefresh({proc = process, timeoutMs = 15000} = {})` (or in `mcpLaunchAdmission.mjs` / `fleetMcpLauncher.mjs`) that sends `{type: 'neo-fleet-mcp-refresh-env', id}` when `typeof proc?.send === 'function'` and resolves with the refreshed `env` object (or `null` when not running under an IPC parent).
2. **`ConnectionService` Token Refresh, Auto-Reconnect & Auth-Rejection Guard (`ai/services/neural-link/ConnectionService.mjs`):**
   - Add `refreshBridgeCredentials()` on `ConnectionService` (using an injectable `this.refreshEnvFn` defaulting to `requestParentEnvRefresh`): when called, if it returns fresh `NEO_FLEET_BRIDGE_TOKEN` (and optional `NEO_NL_PORT`), update `this.bridgeToken` (`process.env.NEO_FLEET_BRIDGE_TOKEN = env.NEO_FLEET_BRIDGE_TOKEN`) so `createBridgeUrl()` uses the fresh token.
   - In `connectToBridge()`:
     - When `ws.on('close', (code, reason))` fires:
       - If `code === 1008`, attach `authenticationFailed = true` to the handshake rejection error.
       - If `this.bridgeSocket === ws` (an established connection closed): set `this.bridgeSocket = null`, clear `this.sessionData.clear()` and `this.activeAgents.clear()`, and if `!this.manualDisconnect`, trigger `this.scheduleReconnect()` (bounded exponential backoff starting at `500` ms, refreshing credentials via `refreshBridgeCredentials()` before reconnecting).
   - In `ensureBridgeAndConnect()`:
     - Before connecting on a reconnect or `manage_connection start` (or if the initial connect fails with `authenticationFailed`), call `await this.refreshBridgeCredentials()` and retry `connectToBridge()` with the refreshed token.
     - Never call `this.spawnBridgeProcess()` when `err?.authenticationFailed` is true OR when the seat is Fleet-token-managed (`Boolean(this.bridgeToken || process.env.NEO_FLEET_BRIDGE_TOKEN)`), because the keyed Bridge is owned and supervised by the Fleet (`settleFleetBridge` / `superviseFleetBridge`), not by an unkeyed child fallback.
   - In `manageConnection({action})`:
     - Set `this.manualDisconnect = true` (and clear any pending reconnect timer) on `action === 'stop'`, and set `this.manualDisconnect = false` before `await this.ensureBridgeAndConnect()` on `action === 'start'`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `admitLaunch` & `runTarget` (`ai/mcp/client/fleetMcpLauncher.mjs`) | `McpLaunchAdmissionService.mjs:55–56`; `mcpLaunchAdmission.mjs:62` | Parent keeps grant in memory, opens `ipc` fd 3 when `refreshEnv` is provided, and answers `neo-fleet-mcp-refresh-env` with freshly redeemed `admittedEnv` (`NEO_FLEET_BRIDGE_TOKEN`) | `stdio: 'inherit'` when `refreshEnv` is omitted; `{ok: false, code, reason}` on refusal | `fleetMcpLauncher.mjs` JSDoc | AC-1 |
| `ConnectionService` token refresh & auto-reconnect (`ai/services/neural-link/ConnectionService.mjs`) | `#732` terminal predicate; `#1203` / `#1212` Bridge supervision | On unexpected Bridge close, clears stale `sessionData`/`activeAgents`, refreshes `NEO_FLEET_BRIDGE_TOKEN` from parent `fleetMcpLauncher`, and reconnects automatically | Manual `manage_connection start` also refreshes token and reconnects | `ConnectionService.mjs` JSDoc | AC-2, AC-3 |
| `ConnectionService#ensureBridgeAndConnect` (`ConnectionService.mjs:555`) | `settleFleetBridge.mjs` (`#1203` / `#1212`) | Never spawns an unkeyed `Bridge.mjs` on `1008` auth rejection or when `NEO_FLEET_BRIDGE_TOKEN` is set | Standalone (unmanaged) mode without `NEO_FLEET_BRIDGE_TOKEN` still spawns local Bridge | `ConnectionService.mjs` JSDoc | AC-4 |

Decision Record impact: `aligned-with` `ADR 0034` / `ADR 0038` (preserves `PROCESS_CONTROL` grant isolation in `mcpLaunchAdmission.mjs:62` while using `McpLaunchAdmissionService#redeem`'s multi-redemption contract).

## Acceptance Criteria

- [ ] AC-1 Unit (`FleetMcpLauncher.spec.mjs`): `runTarget` with `refreshEnv` answers `{type: 'neo-fleet-mcp-refresh-env', id}` over IPC with the newly redeemed `NEO_FLEET_BRIDGE_TOKEN` while `NEO_FLEET_LAUNCH_GRANT` and `NEO_FLEET_LAUNCH_ISSUER` remain absent from the child's environment.
- [ ] AC-2 Unit (`ConnectionService` spec): when a keyed Bridge restarts after the initial `NEO_FLEET_BRIDGE_TOKEN` has expired (`> 1h`), `ConnectionService` refreshes its token via the parent refresh seam, clears stale `sessionData`/`activeAgents`, and automatically reconnects and re-registers on the restarted Bridge.
- [ ] AC-3 Unit (`ConnectionService` spec): `manage_connection({action: 'start'})` after token expiry refreshes `NEO_FLEET_BRIDGE_TOKEN` and connects cleanly; `manage_connection({action: 'stop'})` suppresses auto-reconnect until `start` is called again.
- [ ] AC-4 Unit (`ConnectionService` spec): when `connectToBridge()` is rejected with `1008` (`Authentication required`) or when `NEO_FLEET_BRIDGE_TOKEN` is present, `ensureBridgeAndConnect()` never calls `spawnBridgeProcess()` to spawn an unkeyed Bridge.

## Out of Scope

- Changing `bridgeTokenTtlMs` (`1h`) in `FleetRegistryService.mjs`.
- CLI seats that do not yet launch `neural-link` through `fleetMcpLauncher.mjs`.

## Avoided Traps

- Passing `NEO_FLEET_LAUNCH_GRANT` or `NEO_FLEET_LAUNCH_ISSUER` into the child's `process.env` (`PROCESS_CONTROL` in `mcpLaunchAdmission.mjs:62` explicitly forbids this so tool execution inside the MCP server cannot read the seat's raw capability secret).
- Letting `ensureBridgeAndConnect()` fall back to `spawnBridgeProcess()` on a `1008` close or for a Fleet-token-managed seat, which squats `:8081` with an unkeyed Bridge (`FLEET_BRIDGE_STATES.UNKEYED`).

## Related

- Parent epic: `neomjs/neo-agent-institution#732` (*An FM update never requires closing a harness*)
- Sibling Bridge supervision ticket/PR: `neomjs/neo-agent-brain#1203` / `neomjs/neo-agent-brain#1212`
- Source `defect-note`: `MESSAGE:59608605-51a9-4fae-ad64-a87bfaca76c6` (`@neo-opus-vega`)

## Sweeps (`ticket-create §1`)

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-19T19:26:28Z` (`#1198`, `#1174`, `#1149`, `#1081`, …); no existing ticket for `ConnectionService` reconnect + `fleetMcpLauncher` token refresh.
- A2A in-flight claim sweep: checked latest 30 messages across all read states at `2026-10-19T19:26:30Z`; `@neo-opus-vega` captured `MESSAGE:59608605-51a9-4fae-ad64-a87bfaca76c6` as a `defect-note` and no peer has promoted or claimed it.
- Memory Core rationale sweep: queried `query_raw_memories` for `Neural Link client Bridge restart reconnect bridgeTokenTtlMs fleetMcpLauncher` (`3` results); no prior ticket or conflicting decision.
- Own-assignment sweep: `1` open issue assigned to `@neo-gemini-julia` (`#1081`, PR `#1214` in review); no overlap.
- Structure map (§1c): ran `npm run ai:structure-map -- --files --loc`; edits existing `ai/mcp/client/fleetMcpLauncher.mjs` and `ai/services/neural-link/ConnectionService.mjs`.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb
Retrieval Hint: "Neural Link client Bridge restart reconnect bridgeTokenTtlMs fleetMcpLauncher ConnectionService 1008"

## Timeline

- 2026-10-19T19:27:38Z @neo-gemini-julia added the `bug` label
- 2026-10-19T19:27:39Z @neo-gemini-julia added the `ai` label
- 2026-10-19T19:27:39Z @neo-gemini-julia added the `testing` label
- 2026-10-19T19:27:39Z @neo-gemini-julia added the `architecture` label
- 2026-10-19T19:27:39Z @neo-gemini-julia added the `agent-os` label
- 2026-10-19T19:28:25Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T19:28:30Z @neo-gemini-julia added parent issue #732
- 2026-10-19T19:43:33Z @neo-gemini-julia cross-referenced by PR #1216
### @neo-gemini-julia - 2026-10-19T21:20:22Z

## Session Sunset Handover (`neomjs/neo-agent-brain#1215`)

- **Owner:** `@neo-gemini-julia` (Julia, Gemini 4 Argon)
- **PR:** [`neomjs/neo-agent-brain#1216`](https://github.com/neomjs/neo-agent-brain/pull/1216) (`Resolves #1215`, head [`d995b8ed`](https://github.com/neomjs/neo-agent-brain/pull/1216/commits/d995b8ede32d80dbb31fc0f93db646845c687c62))
- **Status:** **APPROVED** by `@neo-opus-vega` (`5577874480`, cross-family, Round 1, no required actions) with `24/24` CI checks green (`1` skipped) — ready for human merge (`@tobiu`).
- **Post-Merge Validation (`AC-5`, `Residual-Owner: neomjs/neo-agent-institution#732`):** `@neo-opus-vega` noted in review `5577874480` that she will witness the installed outcome on `neomjs/neo-agent-institution#732` at the first Bridge restart after `#1216` ships.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

- 2026-10-20T10:22:15Z @tobiu referenced in commit `ada3661` - "fix(neural-link): reconnect after Bridge restart with fresh parent-redeemed token (#1215) (#1216)"
- 2026-10-20T10:22:16Z @tobiu closed this issue

