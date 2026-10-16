---
id: 1062
title: Keep stdioToStreamableHttp alive on in-flight SSE drops and 404 resets
state: OPEN
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-10-15T23:26:46Z'
updatedAt: '2026-10-15T23:26:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1062'
author: neo-gemini-pro
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
# Keep stdioToStreamableHttp alive on in-flight SSE drops and 404 resets

## Context

Observed on `2026-10-15T23:10:45Z` on seat `@neo-gemini-pro` (Antigravity `2.23.1`), 20 minutes after a clean Fleet seat restart (`22:50:38Z`) where both the Fleet launch admission issuer (`http://127.0.0.1:52363`) and the containerized Memory Core server (`http://127.0.0.1:3102/mc/mcp`) were up and healthy:

1. At `23:10:34Z`, `@neo-gemini-pro` called `add_message` on `neo-mjs-memory-core`.
2. At `23:10:45Z` (11 seconds later), the tool call failed with `Encountered error in tool execution: calling "tools/call": EOF`, and the `stdioToStreamableHttp.mjs --url http://127.0.0.1:3102/mc/mcp` child process (PID `51081`) exited with status `1`.
3. At `23:10:52Z` and `23:12:14Z`, every subsequent `neo-mjs-memory-core` tool call failed in `<1s` with `Encountered error in tool execution: server name neo-mjs-memory-core failed to load: exit status 1` — even though running `/Users/tobiasuhlig/.neo-ai/agents/neo-gemini-pro/harness/antigravity/.neo-fleet-mcp/memory-core.sh` directly succeeded in `3s` against the same live issuer (`127.0.0.1:52363`) and plane (`127.0.0.1:3102/mc/mcp`).

## The Problem

`#1041` updated `ai/mcp/client/stdioToStreamableHttp.mjs` so that `onerror(SSE stream disconnected)` when `pendingRequestIds.size === 0` is logged and survived (`:195-209`). However, `isTerminalRemoteError` (`:191-197`) still treats three remote transport conditions as process-ending fatal errors (`fail(error)` -> `process.exitCode = 1; close({terminate: false})`):

1. `cause === 'sse-disconnect' && pendingRequestIds.size > 0` (`:196`)
2. `cause === 'sse-reconnect-exhausted'` (`:195`, which `@modelcontextprotocol/sdk`'s `DEFAULT_STREAMABLE_HTTP_RECONNECTION_OPTIONS` triggers after just `maxRetries: 2` — `1s` + `1.5s` = `2.5s`)
3. `error?.code === 404` (`SESSION_ENDING_CODES`, when `mc-server` or `kb-server` restarts and its in-memory `TransportService.transports` map no longer holds the old `Mcp-Session-Id`)

Closing `localTransport` (stdio) and exiting the `stdioToStreamableHttp.mjs` process with `exitCode = 1` on those conditions causes three failures across the Fleet:

- **Permanent MCP death in Antigravity:** Unlike Claude Code, Antigravity's `language_server` (`2.23.1`) spawns each configured stdio MCP server once and **never respawns an initialized stdio server that exits with `exit status 1`**. A single `fail(error)` permanently kills `neo-mjs-memory-core` (or `neo-mjs-knowledge-base`) for the rest of the Antigravity window's life, even when the Fleet launch admission issuer is healthy.
- **Unanswered in-flight JSON-RPC requests (`EOF`):** In `@modelcontextprotocol/sdk/dist/esm/client/streamableHttp.js:401-406`, `send()` resolves as soon as the HTTP `200 OK` (`Content-Type: text/event-stream`) headers arrive, and `_handleSseStream` reads the SSE response body asynchronously while the standalone `GET /mc/mcp` SSE stream (`:390`) is also open. When `_handleSseStream` raises `onerror(SSE stream disconnected)` while `pendingRequestIds.size > 0`, `send()` has already resolved, so `stdioToStreamableHttp.mjs:239` never catches it; instead, `onRemoteError` calls `fail(error)` and closes stdout without sending a JSON-RPC error response for `pendingRequestIds`, producing `calling "tools/call": EOF` (or `Connection closed` in Claude Desktop).
- **Avoidable bridge death across plane restarts & adopted seats:** When `mc-server` or `kb-server` restarts (~5–30s), the standalone `GET` stream exhausts its 2 retries in 2.5s (`sse-reconnect-exhausted`) and subsequent POSTs receive `404 Session not found`. Because `stdioToStreamableHttp.mjs` exits `1` instead of answering any pending request ids with a JSON-RPC `InternalError` and re-initializing a fresh `StreamableHTTPClientTransport` session when the plane returns, every seat loses its bridge process (and adopted seats whose Fleet Manager was restarted fail launch admission on respawn, per `#980` / `neomjs/neo-agent-institution#674`).

## The Architectural Reality

- `ai/mcp/client/stdioToStreamableHttp.mjs:175-197`:
  - `fail(error)` sets `terminalErrorReported = true`, writes `RUNTIME_FAILURE_MESSAGE` (`process.exitCode = 1` via `:350-353`), and calls `close({terminate: false})` without replying to `pendingRequestIds`.
  - `isTerminalRemoteError(error)` returns `true` for `!initialized || SESSION_ENDING_CODES.has(error?.code) || cause === 'sse-reconnect-exhausted' || (cause === 'sse-disconnect' && pendingRequestIds.size > 0)`.
- `ai/mcp/client/stdioToStreamableHttp.mjs:229-260`:
  - `localTransport.onmessage` records `initializeRequestId` and `pendingRequestIds.add(message.id)`, but does not retain the `initialize` request parameters (`protocolVersion`, `capabilities`, `clientInfo`) needed to transparently re-initialize a fresh `StreamableHTTPClientTransport` when the upstream HTTP session drops or returns `404 Session not found`.
- `ai/mcp/server/shared/services/TransportService.mjs:290-296`:
  - Stores active `StreamableHTTPServerTransport` instances in memory (`this.transports.get(sessionId)`) and returns `404 { error: 'Session not found' }` after a container restart.

## The Fix

1. **Never close `localTransport` leaving `pendingRequestIds` unanswered on remote stream drops:** When a remote SSE disconnect or session reset occurs while `pendingRequestIds.size > 0`, drain `pendingRequestIds` by sending a JSON-RPC `ErrorCode.InternalError` response (`${REQUEST_FAILURE_MESSAGE} Cause: ${cause}.`) on `localTransport` for each pending request ID instead of closing `localTransport` (`EOF`) and exiting with `exitCode = 1`.
2. **Recover/re-initialize the remote HTTP session instead of exiting on `sse-disconnect`, `sse-reconnect-exhausted`, or `http-404` (`Session not found`):**
   - Cache the client's `initialize` request payload once initialization succeeds.
   - When the remote session is lost (`http-404` after a plane container restart, `sse-reconnect-exhausted`, or an in-flight `sse-disconnect`), close only the broken `remoteTransport` instance, mark the upstream session as needing re-initialization, and transparently create + handshake a fresh `StreamableHTTPClientTransport` on the next client request (while keeping `401` / `403` authentication/authorization refusals and pre-initialization failures terminal).
3. Extend `test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs` to cover:
   - In-flight `sse-disconnect` while a request is pending returns a JSON-RPC `InternalError` for that request ID and keeps the bridge alive for subsequent requests (never `EOF` / process exit).
   - `sse-reconnect-exhausted` and `http-404` (`Session not found` after plane restart) recover by re-initializing a fresh remote transport on the next request while keeping `401` / `403` terminal.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `bridgeTransports` / `startBridge` | `ai/mcp/client/stdioToStreamableHttp.mjs:133,311` | Answers pending request IDs with JSON-RPC `InternalError` on in-flight `sse-disconnect` / remote session loss and re-initializes the remote `StreamableHTTPClientTransport` on `sse-reconnect-exhausted` or `http-404` without exiting the stdio process | Terminal close (`exitCode = 1`) on pre-init failure, `401`, `403`, or local stdio error | JSDoc in `ai/mcp/client/stdioToStreamableHttp.mjs` | `test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs` |

Decision Record impact: none

## Acceptance Criteria

- [ ] AC-1: When `onerror(SSE stream disconnected)` fires while one or more requests are in `pendingRequestIds`, the bridge answers each pending request ID on `localTransport` with a JSON-RPC `InternalError` naming `Cause: sse-disconnect.` and keeps `localTransport` open so subsequent requests succeed.
- [ ] AC-2: When `sse-reconnect-exhausted` or post-initialization `http-404` (`Session not found`) occurs, the bridge retires the stale remote transport and transparently re-initializes a fresh remote transport on the next request without exiting the stdio process.
- [ ] AC-3: Pre-initialization failures, `http-401`, `http-403`, `unexpected-content-type`, and `localTransport.onerror` remain terminal and close the bridge with `process.exitCode = 1`.
- [ ] AC-4: Unit tests in `test/playwright/unit/ai/mcp/client/StdioToStreamableHttp.spec.mjs` verify AC-1, AC-2, and AC-3 with no raw remote error text or bearer material written to stderr.

## Out of Scope

- Fleet launch admission issuer succession across Fleet Manager restarts (`#980` / `neomjs/neo-agent-institution#674`).
- Modifying `@modelcontextprotocol/sdk` internals.

## Avoided Traps

- Closing `localTransport` and exiting `1` while assuming the host IDE will respawn the stdio server: Antigravity (`language_server` 2.23.1) permanently marks an exited stdio MCP server as `failed to load: exit status 1` and never respawns it.
- Swallowing `401` / `403` auth refusals: credential revocations must still terminate the bridge immediately.

## Related

- `#1041` (added cause vocabulary and survived idle `sse-disconnect`, but intentionally kept in-flight `sse-disconnect`, `sse-reconnect-exhausted`, and `404` terminal)
- `#980` / `neomjs/neo-agent-institution#674`
- `#1035` / `#1044`

unowned-rationale: Filed as a concrete root-cause capture after observing `calling "tools/call": EOF` -> `server name neo-mjs-memory-core failed to load: exit status 1` on `@neo-gemini-pro` at `2026-10-15T23:10:45Z`; ready for pickup.

Sweeps (`2026-10-15T23:18Z`):
- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-15T23:18Z`; `#1041` is closed and scoped in-flight `sse-disconnect` / `sse-reconnect-exhausted` / `404` as terminal; no open duplicate exists.
- A2A in-flight claim sweep: checked `list_messages({status: 'all', limit: 15})`; Grace recorded `defect-note` `MESSAGE:625ddf1d-ab21-4939-88f2-93e184054140` on adopted-seat issuer succession (`#980`), no competing claim on `stdioToStreamableHttp.mjs` in-flight SSE disconnect / `exit status 1` permanence in Antigravity.
- Memory Core rationale sweep: `query_raw_memories` (`"stdioToStreamableHttp sse-disconnect exit status 1 generateAntigravityMcpFiles PATH gh"`); no conflicting decision.
- Own-assignment sweep: only `#1056` assigned to `@neo-gemini-pro`.
- Structure map: `ai/mcp/client/` owns `stdioToStreamableHttp.mjs`.

Origin Session ID: 14f10506-4e8f-40bb-96be-69dac6effa98
Retrieval Hint: "stdioToStreamableHttp sse-disconnect pendingRequestIds exit status 1 Antigravity EOF"


## Timeline

- 2026-10-15T23:26:47Z @neo-gemini-pro added the `bug` label
- 2026-10-15T23:26:48Z @neo-gemini-pro added the `ai` label
- 2026-10-15T23:26:48Z @neo-gemini-pro added the `model-experience` label
- 2026-10-15T23:26:48Z @neo-gemini-pro added the `agent-os` label
- 2026-10-15T23:40:11Z @neo-gpt-sophie cross-referenced by #12

