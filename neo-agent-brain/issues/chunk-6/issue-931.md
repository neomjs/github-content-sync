---
id: 931
title: healthcheck session.currentId names the seat that filled the cache
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-08T09:24:06Z'
updatedAt: '2026-10-08T09:35:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/931'
author: neo-opus-ada
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
# healthcheck session.currentId names the seat that filled the cache

## Context

On the shared plane `neo-local-canonical`, a seat that calls `healthcheck` can receive another seat's Memory Core session id.

Observed 2026-10-08:
- Ada's call at 08:52:36Z returned `session.currentId` `3cef6883-7cf3-4596-b956-a16e37ec29cd`.
- Grace's own `healthcheck`, 78 s later, returned the same id. Her turns sit in that session only because she then passed it to `add_memory` explicitly. So whose request bound `3cef6883` is unproven: a third caller's connection, or the process fallback `_legacySessionId`. (Corrected 2026-10-08 after Grace's datum; the first version named her as the owner.)
- Ada's own `add_memory` echoed `58ad7fe7-062f-4a99-a92c-e5867f6ba8aa`.

Ada had stamped the wrong id as `Origin Session ID` on three reviews (neomjs/neo-agent-brain#928, neomjs/neo#19464, neomjs/neo#19473), and posted a correction beside each one. Triage promoted the defect-note `MESSAGE:1b1f2a66` to this ticket.

## The Problem

`healthcheck` caches its healthy payload process-wide for five minutes, and `session.currentId` goes into the cache with everything else, although it is a per-request value. A seat calling within five minutes of another seat's full check gets that seat's id.

Both cache hits do it:
- `freshObservability: false` returns the cached payload unchanged.
- The default request-fresh path spreads `...cachedHealth` and refreshes only `timestamp`, `runtimeFreshness` and the collection counts.

The openapi field reads "The ID of the current active session", so a caller takes it as its own and stamps it as provenance.

## The Architectural Reality

The three files are identical at the deployed revision `2d839fc1` and at `dev` `88c9d8b9`. The owning folder is `ai/services/memory-core` (35 files in the structure map). No new file.

- The `SessionService.currentSessionId` getter returns `RequestContextService.getSessionId() || this._legacySessionId`: the request-bound session first (AsyncLocalStorage, `ai/mcp/server/shared/services/RequestContextService.mjs`), then a process-wide fallback.
- `HealthService.#performHealthCheck` writes `session: {currentId: SessionService.currentSessionId}` into the payload. `healthcheck()` keeps healthy payloads in `#cachedHealth` for `#cacheDuration` (5 min).
- A cache hit with `freshObservability: false` returns `this.#applyEmbeddingWriteCanary(this.#cachedHealth)`.
- A default cache hit goes through `#buildRequestFreshCachedHealth`, which returns `{...cachedHealth, timestamp, runtimeFreshness, database}`.
- `add_memory` is unaffected, because `MemoryService` reads the same getter inside the caller's own request context.

Design authority: the getter's request-bound order, and the JSDoc of `#buildRequestFreshCachedHealth`: "Direct operator healthcheck calls still need request-time observability".

## The Fix

`healthcheck()` attaches `session.currentId` from `SessionService.currentSessionId` at serve time on every return path: the full check and both cache hits. The cache can no longer carry one caller's id to another.

The openapi description names the field as the calling client's session: its `Mcp-Session-Id`, else the process session when the request carries none.

Add a control to `test/playwright/unit/ai/services/memory-core/HealthService.spec.mjs`. Run two `RequestContextService.run({sessionId})` contexts; the second caller's cache hit returns its own id, under both `freshObservability` values.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `healthcheck` → `session.currentId` | the `SessionService.currentSessionId` getter (request-bound first) | the calling client's session on every return path, cached or not | the process session when the request carries none | openapi field description | unit, red first |

## Acceptance Criteria

- [ ] On a cache hit, `session.currentId` is the caller's own request-bound session for both `freshObservability` values. Red first on today's code.
- [ ] A full check and a later cache hit by the same caller report the same id; the rest of the cached payload is unchanged.
- [ ] The openapi description names the field as the calling client's session.
- [ ] **Post-merge, on the shared plane:** two seats calling `healthcheck` within five minutes each read their own id, the same one their next `add_memory` echoes.

## Out of Scope

- The process-wide `_legacySessionId` fallback for clients without an `Mcp-Session-Id`.
- `Origin Session ID` values already published; corrections are posted where known.
- The cache duration, and which probes are cached.

## Avoided Traps

- **Disabling the cache.** It protects the `ensureHealthy()` gate from repeated Chroma probes.
- **Dropping the field.** Agents use it to find their own session, so it must be right, not gone.

## Related

neomjs/neo-agent-brain#928 · neomjs/neo#19464 · neomjs/neo#19473 · defect-note `MESSAGE:1b1f2a66`. Root cause, not this defect: D#19401 (Memory sessions survive MCP reconnects). The Memory Core session is the MCP connection's id, so a reconnect or a container restart mints a new one, and an agent has to ask the server which session it is in. This ticket only stops the healthcheck cache from handing one caller's id to another, which stays wrong even once ids are stable.

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-08T09:23:21Z, no equivalent. Exact search across the org for "healthcheck session", "Origin Session ID healthcheck", "session.currentId", "cachedHealth" and "currentSessionId" found no open equivalent.
A2A sweep: the latest 30 messages in all read states hold no claim on this scope.
MC sweep: the problem's nouns (another agent's session id from healthcheck, wrong Origin Session ID, cached currentId) surfaced Sophie's 2026-10-05 reconnect analysis behind D#19401, and no prior decision on this cache.
Own-assignment sweep: 15 open, none overlapping.

Origin Session ID: 58ad7fe7-062f-4a99-a92c-e5867f6ba8aa
Retrieval Hint: "healthcheck session currentId cached payload wrong seat Origin Session ID provenance"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-10-08T09:24:06Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-08T09:24:08Z @neo-opus-ada added the `bug` label
- 2026-10-08T09:24:08Z @neo-opus-ada added the `ai` label
- 2026-10-08T09:24:08Z @neo-opus-ada added the `agent-os` label
- 2026-10-08T09:52:22Z @neo-opus-ada cross-referenced by #932
- 2026-10-08T12:22:05Z @neo-opus-ada cross-referenced by PR #935

