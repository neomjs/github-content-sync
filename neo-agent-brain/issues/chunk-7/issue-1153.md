---
id: 1153
title: Bridge evicts an agent's still-open socket when a newer socket closes
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-18T21:04:44Z'
updatedAt: '2026-10-18T22:27:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1153'
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
closedAt: '2026-10-18T22:27:57Z'
---
# Bridge evicts an agent's still-open socket when a newer socket closes

## Context

Promoted from `@neo-opus-vega`'s `defect-note` (`MESSAGE:1f8e39fc-dd3f-4662-af09-1a57d6a63005`, `2026-10-18T19:43:55Z`) and lane split (`MESSAGE:6ff370b3-badd-4178-b3ce-7cfc3718b57f`, under `neomjs/neo-agent-institution#732`):

> `defect-note: Brain neural-link ConnectionService is wrong for a Codex seat's NL server — one process (neo-gpt-sophie) opened up to three concurrent Bridge sockets and flapped 15× in 46 s (19:37:47–19:38:33Z), ending with no registered socket while the client still reports connected=true, so app-command replies are dropped and time out`
> `Same session answered instantly from another seat. Bridge has #1023's close-path fix (PR #1054). Bridge INFO logs carry no disconnect reason; that's a second gap.`

**Design authority:** `ai/mcp/server/neural-link/Bridge.mjs:243-246` (`registerAgent` multi-session contract from `#1023` / PR `#1054`) and `ai/services/neural-link/ConnectionService.mjs:198-199, 478-507` (`ensureBridgeAndConnect` client connection lifecycle).

## The Problem

Three compounding defects produce the state observed in `neo-gpt-sophie`'s session:

1. **Closing a newer socket under the same `agentId` evicts a still-open older socket from `Bridge.agents` (`Bridge.mjs:252, 263-269, 526-531`):**
   - In fleet mode (`NEO_FLEET_BRIDGE_PUBLIC_KEY`), every socket opened by a seat's Neural Link MCP server(s) authenticates under the same token-signed `agentId` (`verifiedId`, `:221-227`).
   - `#1023` (PR `#1054`) guarded `ws.on('close')` with `if (this.agents.get(id) === ws) { this.agents.delete(id); ... }`, which protects `newer` when `older` closes first.
   - However, when `older` (`s1`, held by the seat's long-lived `ConnectionService`) stays open while a second socket `newer` (`s2`, opened by a concurrent Codex MCP session, probe, or overlapping start) connects and then closes **first**:
     - While `s2` is open, `this.agents.set(id, s2)` overwrites `s1`, so `broadcastToAgents` (`:528`) stops delivering `app_message` RPC replies to `s1`.
     - When `s2` closes, `this.agents.get(id) === s2` is `true`, so `this.agents.delete(id)` **deletes `id` from `Bridge.agents` completely** even though `s1` is still open (`s1.readyState === WebSocket.OPEN`, client `bridgeConnected: true`).
     - Subsequent RPC calls sent over `s1` reach `handleAgentMessage` (`:458`) with `this.agents.get(agentId) === undefined`, forward to the App, and the App's reply is dropped by `broadcastToAgents` (`:528`) because `id` is no longer in `Bridge.agents` — causing every call on `s1` to hang until the 10 s RPC timeout.
2. **`ConnectionService.ensureBridgeAndConnect()` opens duplicate sockets when already connected or when called concurrently (`ConnectionService.mjs:478-507`):**
   - `Server.boot()` (`ai/mcp/server/neural-link/Server.mjs:190-206`) connects the stdio MCP transport *before* awaiting `ConnectionService.ensureBridgeAndConnect()`.
   - `ensureBridgeAndConnect()` (`:478`) neither short-circuits when `this.bridgeSocket?.readyState === WebSocket.OPEN` nor coalesces concurrent in-flight calls, so an overlapping `manage_connection({ action: 'start' })` or repeat `ensureBridgeAndConnect()` call opens a second WebSocket from the same singleton without closing the first.
3. **WebSocket `close` logs omit close code and reason (`Bridge.mjs:260, 421` and `ConnectionService.mjs:451`):**
   - All three `ws.on('close')` handlers ignore `(code, reason)`, leaving operator logs with bare `Bridge: Agent disconnected [id]` / `Disconnected from Neural Link Bridge` lines and no close attribution.

## The Architectural Reality

- **`ai/mcp/server/neural-link/Bridge.mjs`:**
  - `this.agents` (`Map<agentId, WebSocket>`, `:45`) is inspected directly by existing callers and unit tests (`Bridge.spec.mjs:67, 103, 175, 293`), which expect `Bridge.agents.get(id)` to hold the current active `WebSocket` for `id`.
  - Pairing `this.agents` with a per-agent live socket set (`agentSockets = new Map()`, `Map<agentId, Set<WebSocket>>`) preserves `Bridge.agents.get(id)` as the newest live socket while ensuring:
    1. When `newer` closes while `older` is still registered and open, `this.agents.set(id, older)` promotes the most-recent remaining open socket instead of deleting `id`, and no false `agent_disconnected` frame is sent to Agents.
    2. While multiple sockets for `id` are simultaneously open, `broadcastToAgents` (`:526`) delivers frames (`app_message`, etc.) to every open socket for `id`, and `handleAgentMessage` (`:449`) uses the sending socket's own `ws.sessionId`.
- **`ai/services/neural-link/ConnectionService.mjs`:**
  - Direct `connectToBridge()` (`:376`) is tested as a low-level primitive in `ConnectionService.spec.mjs:313-378` (`a retired socket close cannot reject calls on its replacement socket`). Keeping raw `connectToBridge()` unchanged while making `ensureBridgeAndConnect()` (`:478`) short-circuit when `this.bridgeSocket?.readyState === WebSocket.OPEN` and coalesce concurrent in-flight calls via `#ensureConnectPromise` prevents duplicate sockets from `Server.boot()` / `manage_connection({ action: 'start' })` without altering low-level replacement-socket semantics.

## The Fix

1. **`ai/mcp/server/neural-link/Bridge.mjs`:**
   - Add `agentSockets = new Map()` (`Map<agentId, Set<WebSocket>>`) alongside `this.agents`, reconciled against `this.agents` so direct test map resets (`Bridge.agents = new Map()`) stay hermetic.
   - In `registerAgent(id, ws)`: add `ws` to `agentSockets.get(id)`, set `this.agents.set(id, ws)`, and bind `ws.on('message', (data) => this.handleAgentMessage(id, data, ws))`.
   - In `ws.on('close', (code, reason) => ...)`: log `code` and `reason`, remove `ws` from `agentSockets.get(id)`, and if another live socket remains in the set for `id`, promote the most-recent remaining live socket into `this.agents.set(id, remainingWs)` without broadcasting `agent_disconnected` to Agents; only when no live socket remains for `id`, delete `id` from `this.agents` and `agentSockets` and broadcast `agent_disconnected` to Agents. Always broadcast `{ type: 'agent_disconnected', agentId: id, sessionId: ws.sessionId }` to Apps.
   - In `broadcastToAgents(payload)`: send `data` to every live socket across `agentSockets` and `this.agents` (deduplicated per `WebSocket` instance).
   - In `handleAgentMessage(agentId, data, senderWs)`: resolve `const agentWs = senderWs || this.agents.get(agentId)` so each socket stamps its own `sessionId` and receives its own target-not-found error replies.
   - In `registerApp` (`:421`) and `stopServer` (`:120`): log `(code, reason)` on app disconnect, and terminate all sockets in `agentSockets` on `stopServer()`.
2. **`ai/services/neural-link/ConnectionService.mjs`:**
   - In `ensureBridgeAndConnect()` (`:478`): return immediately when `this.bridgeSocket?.readyState === WebSocket.OPEN`, and coalesce concurrent calls onto `#ensureConnectPromise` (cleared in `finally`).
   - In `connectToBridge()` `ws.on('close', (code, reason) => ...)` (`:451`): include `(code, reason)` in the disconnect warning log.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `Bridge.registerAgent` & `Bridge.broadcastToAgents` (`ai/mcp/server/neural-link/Bridge.mjs`) | `Bridge.mjs:243-246` (`#1023` / PR `#1054`) | Tracks all live sockets per `agentId`; closing a newer socket promotes the most-recent still-open socket in `Bridge.agents` instead of deleting `id`; all open sockets for `id` receive `broadcastToAgents` replies and stamp their own `ws.sessionId` | Last socket closing deletes `id` and broadcasts `agent_disconnected` to Agents as before | Method JSDoc | `Bridge.spec.mjs` RED-first unit arms |
| `ConnectionService.ensureBridgeAndConnect` (`ai/services/neural-link/ConnectionService.mjs`) | `ConnectionService.mjs:478-507` | No-ops when `this.bridgeSocket?.readyState === WebSocket.OPEN` and coalesces concurrent in-flight `ensureBridgeAndConnect()` calls onto one promise | Proceeds with connect/spawn when socket is absent or closed | Method JSDoc | `ConnectionService.spec.mjs` RED-first unit arms |
| Disconnect logging (`Bridge.mjs:260, 421`, `ConnectionService.mjs:451`) | `@neo-opus-vega` defect-note `MESSAGE:1f8e39fc` | Logs WebSocket close `code` and `reason` on agent, app, and client disconnect | `code=none, reason=none` when omitted | Inline | Unit verification in `Bridge.spec.mjs` |

**Decision Record impact:** `none` (bug fix within existing Neural Link Bridge and ConnectionService contracts; aligned with `ADR-0040`).

## Acceptance Criteria

- [ ] AC-1: In `Bridge.mjs`, when `older` and `newer` sockets are registered under the same `agentId` and `newer` closes first while `older` remains open, `Bridge.agents.get(agentId)` retains `older`, no `agent_disconnected` is broadcast to Agents, `older` continues receiving `app_message` broadcasts, and `handleAgentMessage` from `older` stamps `older.sessionId` (unit spec in `Bridge.spec.mjs`, RED on `dev`).
- [ ] AC-2: While multiple sockets are simultaneously open under the same `agentId`, `broadcastToAgents` delivers `app_message` frames to all open sockets for that `agentId` (unit spec in `Bridge.spec.mjs`, RED on `dev`).
- [ ] AC-3: In `ConnectionService.mjs`, `ensureBridgeAndConnect()` is a no-op when `bridgeSocket` is already `WebSocket.OPEN` and coalesces concurrent in-flight `ensureBridgeAndConnect()` calls into a single `connectToBridge()` execution (unit spec in `ConnectionService.spec.mjs`, RED on `dev`).
- [ ] AC-4: `Bridge.mjs` (agent and app `close` handlers) and `ConnectionService.mjs` (bridge socket `close` handler) log the WebSocket close `code` and `reason`.
- [ ] AC-5: All existing and new unit tests in `Bridge.spec.mjs` and `ConnectionService.spec.mjs` pass.

## Out of Scope

- Bridge token TTL renewal after 1 hour (`#1145`, owned by `@neo-opus-grace`).
- Continuity handoff survivor-validation policy (`#1143`, owned by `@neo-opus-vega`).

## Avoided Traps

- **Terminating `older` in `Bridge.registerAgent` (like `registerApp` does at `:410`):** As documented at `Bridge.mjs:243-246` (`#1023`), a seat running two sessions (or a transient MCP tool child alongside a long-lived session) legitimately holds two concurrent connections under its `agentId`; terminating `older` would cause the two processes to ping-pong-kill each other's sockets.
- **Closing `this.bridgeSocket` inside raw `connectToBridge()`:** `ConnectionService.spec.mjs:313-378` tests low-level replacement-socket coexistence by calling `connectToBridge()` twice before closing the first socket; guarding at `ensureBridgeAndConnect()` preserves that low-level contract while preventing duplicate sockets on the boot/`manage_connection` path.

## Related

- Promoted from `defect-note` `MESSAGE:1f8e39fc-dd3f-4662-af09-1a57d6a63005` (`@neo-opus-vega`).
- Part of Fleet Manager stabilization (`neomjs/neo-agent-institution#732`, lane split `MESSAGE:6ff370b3-badd-4178-b3ce-7cfc3718b57f`).
- Extends `#1023` / PR `#1054` (`Bridge — two agent sockets under one id`).

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T21:00:44Z` (`sort: 'created'`, newest `#1151`); no duplicate found (`#1145` is Bridge token 1h expiry).
A2A claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T21:00:47Z`; `@neo-opus-vega`'s lane split (`MESSAGE:6ff370b3`) leaves this defect-note open for self-selection, with no competing claim.
Memory Core rationale sweep: `query_raw_memories("Bridge ConnectionService agents delete concurrent socket neural-link")` at `2026-10-18T21:00:54Z` confirmed `#1023` (PR `#1054`) handled only `older` closing after `newer`, not `newer` closing while `older` remains open.
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-pro` in `neomjs/neo-agent-brain`.
Agent OS structure map: existing files `ai/mcp/server/neural-link/Bridge.mjs` and `ai/services/neural-link/ConnectionService.mjs` (no new `.mjs` files).

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb

Retrieval Hint: `query_raw_memories("Bridge registerAgent concurrent sockets newer close evicts older ConnectionService ensureBridgeAndConnect")`


## Timeline

- 2026-10-18T21:04:44Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-18T21:04:45Z @neo-gemini-julia added the `bug` label
- 2026-10-18T21:04:45Z @neo-gemini-julia added the `ai` label
- 2026-10-18T21:04:46Z @neo-gemini-julia added the `agent-os` label
- 2026-10-18T21:13:26Z @neo-gemini-julia cross-referenced by PR #1154
- 2026-10-18T21:36:38Z @neo-gemini-julia referenced in commit `0c8aaee` - "fix(ai): remove test-double liveness branch and agents map reconciliation in Bridge (#1153)"
- 2026-10-18T22:27:57Z @tobiu referenced in commit `2d62f2e` - "fix(ai): Bridge evicts an agent's still-open socket when a newer socket closes (#1153) (#1154)

* fix(ai): preserve still-open Bridge agent sockets and coalesce connects (#1153)

* fix(ai): remove test-double liveness branch and agents map reconciliation in Bridge (#1153)"
- 2026-10-18T22:27:58Z @tobiu closed this issue
- 2026-10-18T23:33:05Z @neo-fable-clio cross-referenced by #1144

