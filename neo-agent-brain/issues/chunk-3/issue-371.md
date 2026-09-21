---
id: 371
title: 'Neural Link: route history rejects UUID window ids, start hides spawn cause'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-18T20:30:10Z'
updatedAt: '2026-09-19T15:17:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/371'
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
closedAt: '2026-09-19T15:17:02Z'
---
# Neural Link: route history rejects UUID window ids, start hides spawn cause

## Context

Found 2026-09-18 while driving a live Portal (`localhost:8080`) through the Neural Link during an exploration of structured-decision models for runtime UI edits. Two tools failed on first contact, and both errors pointed away from the cause.

Sweeps: live latest-20 open (2026-09-18T20:22Z, re-checked 20:29Z) and `state:all` searches for `route history windowId` and `bridge spawn cwd`: no equivalent. #16 owns the harness launch config and stays separate; the Claude Desktop measurement is posted there.

## The Problem

**1. `get_route_history` fails every call.** MCP output validation rejects the response: `Structured content does not match the tool's output schema: data/windowId must be integer`. `openapi.yaml` types this response's `windowId` as `integer`, the only `windowId` in the file typed that way; live window ids are UUID strings (`d643259a-…`). The declaration is identical at origin/dev `d5ae3e8`.

**2. `manage_connection {action: 'start'}` reports the socket, not the spawn.** With no bridge running and a `--cwd` whose `package.json` lacks `ai:server-neural-link`:
- `ensureBridgeAndConnect()` spawns `npm run ai:server-neural-link`, and npm exits 1 (`Missing script`).
- The child's `exit` listener records `lastSpawnFailure = 'BRIDGE_EXIT_1'`.
- The retry `connectToBridge()` throws, and the tool returns `connect ECONNREFUSED 127.0.0.1:8081`.

The recorded cause reaches only `healthcheck` (`spawnFailure`), and `BRIDGE_EXIT_1` names neither the missing script nor the `--cwd` rule. `mcp-server.mjs` passes `--cwd` through as `bridgeCwd` unchecked, although `BRIDGE_NPM_SCRIPT` is exported, per its docblock, so an entrypoint can validate its cwd.

**3. `execute_dock_operation` describes a stale vocabulary.** Its description enumerates 8 operations; a live Workstation workspace (`get_dock_topology`, 2026-09-18) accepts 17. Among the missing are `moveNode`, `resizeEdgeZone`, `setActiveItem`, `transferItem` and `transferNode`. An agent that plans from the description never picks them.

## The Architectural Reality

- `ai/mcp/server/neural-link/openapi.yaml`: `operationId: get_route_history`, response `properties.windowId`.
- `ai/services/neural-link/ConnectionService.mjs`: `manageConnection`, `ensureBridgeAndConnect`, `spawnBridge` (its `exit` listener), `BRIDGE_NPM_SCRIPT`.
- `ai/services/neural-link/HealthService.mjs`: exposes `spawnFailure`.
- `ai/mcp/server/neural-link/mcp-server.mjs`: the `--cwd` option, passed through as `bridgeCwd`.
- ADR-0040, `agentosRuntimeRoot` row: `--cwd` names the Agent OS runtime root, never the target workspace.

## The Fix

1. `get_route_history` response: `windowId: {type: string}`.
2. `manageConnection('start')`: when the spawn attempt recorded a failure, fail with it. Name the code, the script, the cwd and ADR-0040's rule instead of rethrowing the socket error.
3. Check `--cwd` once at entry: its `package.json` must define `BRIDGE_NPM_SCRIPT`. If it does not, `healthcheck` reports it before any spawn, and the spawn refuses with the same message.
4. `execute_dock_operation`: drop the enumerated list from the description and point at `get_dock_topology`, which already returns the live vocabulary.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `get_route_history` output `windowId` | `openapi.yaml` + the App Worker's UUID window ids | string | — | `openapi.yaml` | unit arm validates a UUID-windowId payload against the declared schema |
| `manage_connection` `start` error | `spawnBridge` attribution comments + ADR-0040 | the error carries `lastSpawnFailure`, script and cwd | the socket error only when the spawn succeeded and connect still fails | JSDoc | red-first unit arm with a spawn stub that exits 1 |
| `--cwd` entry check | `BRIDGE_NPM_SCRIPT` docblock + ADR-0040 | `healthcheck` reports an invalid `--cwd` before any spawn | an absent `--cwd` keeps today's deferral | JSDoc | unit arm: a cwd without the script yields the finding |

## Acceptance Criteria

- [ ] No `windowId` in `ai/mcp/server/neural-link/openapi.yaml` is typed `integer`, and a unit arm validates a route-history payload with a UUID window id against the output schema.
- [ ] `manage_connection start` with a spawn that exits non-zero fails with an error carrying the spawn failure and the `--cwd` rule. Red-first arm.
- [ ] A `--cwd` whose `package.json` lacks `ai:server-neural-link` is reported by `healthcheck` before the first spawn attempt.
- [ ] The `execute_dock_operation` description names no operation list of its own and points at `get_dock_topology`'s `operations`.

## Out of Scope

- Harness launch configs (Codex, Claude Desktop): #16.
- Bridge internals.

## Related

- #16: the launch-config half of the same failure.
- neomjs/neo#8924: the precedent for one path contract across the Neural Link readers.

Decision Record impact: aligned-with ADR-0040 (enforces its `--cwd` row at the entrypoint).

unowned-rationale: filed from a live exploration while the reporter's lane is the review of neomjs/neo#18938. One Brain PR; claimable.

Retrieval Hint: "manage_connection ECONNREFUSED BRIDGE_EXIT_1 cwd", "get_route_history windowId integer UUID"
Origin Session ID: fc012fb9-612e-431f-b2d1-115e3b895a32


## Timeline

- 2026-09-18T20:30:11Z @neo-opus-ada added the `bug` label
- 2026-09-18T20:30:11Z @neo-opus-ada added the `ai` label
- 2026-09-18T20:30:12Z @neo-opus-ada added the `agent-os` label
- 2026-09-18T20:30:28Z @neo-opus-ada cross-referenced by #372
- 2026-09-18T20:30:45Z @neo-opus-ada cross-referenced by #18939
- 2026-09-18T20:30:58Z @neo-opus-ada cross-referenced by #16
- 2026-09-19T13:00:19Z @neo-opus-ada cross-referenced by #18956
- 2026-09-19T13:38:39Z @neo-fable assigned to @neo-fable
- 2026-09-19T13:51:42Z @neo-fable cross-referenced by PR #376
- 2026-09-19T15:17:02Z @tobiu referenced in commit `0a46e42` - "Merge pull request #376 from neomjs/feature/371-neural-link-route-history-and-start

fix(neural-link): route history accepts UUID window ids, and a failed start names its spawn (#371)"
- 2026-09-19T15:17:03Z @tobiu closed this issue

