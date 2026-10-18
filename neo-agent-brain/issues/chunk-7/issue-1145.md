---
id: 1145
title: 'A Codex seat''s Bridge token expires an hour after Start, ending its NL'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T19:31:18Z'
updatedAt: '2026-10-18T20:40:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1145'
author: neo-opus-vega
commentsCount: 2
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
# A Codex seat's Bridge token expires an hour after Start, ending its NL

## Context

On 2026-10-18 at 19:21Z the host Bridge was keyed (#1144). Since then it verifies every agent token: signature, plus `Date.now() < expiresAt` (`ai/mcp/server/neural-link/verifyBridgeToken.mjs:18`). Before that, the Bridge ran unkeyed (legacy) and never checked expiry.

**Observed, read-only:**
- A Codex seat's Neural Link row runs `neural-link/mcp-server.mjs` directly. There is no launch admission. The token arrives through `env_vars = [… "NEO_FLEET_BRIDGE_TOKEN" …]` from the Codex process env, set once at the seat's Start (`FleetLifecycleService.mjs:1044–1045`, `mintBridgeToken`).
- `FleetRegistryService.bridgeTokenTtlMs = 60 * 60 * 1000` (:373).
- Claude Desktop and Antigravity seats go through `fleetMcpLauncher`. Their bridge-token owner renews a fresh token on every launch, so they are not affected.

## The Problem

Every Codex seat's Neural Link now works for at most one hour after the seat starts. After that, a new Codex session and any reconnect present an expired token, the Bridge refuses it, and Neural Link is gone until the operator restarts the seat. Nothing on the card says so.

## The Architectural Reality

- Static carriage: Codex `config.toml` rows plus the process env (#1136 / #1137 own the row convergence).
- Renewal exists for launched rows only: `createBridgeTokenOwner(...).resolve` → `registry.renewBridgeToken(agentId, authority)` (`ai/services/fleet/launchCredentialOwners.mjs`).
- Expiry enforcement: `verifyBridgeToken` (above); the TTL is a class field (`FleetRegistryService.mjs:373`).

Design authority: the operator, 2026-10-18 (neomjs/neo-agent-institution#732: the operator never types a command or restarts to keep a seat working), and #13172's token design, which gives a short TTL *with renewal*.

## The Fix

Codex seats get a fresh token per Neural Link launch, as launched rows already do. Either move the Codex NL row onto `fleetMcpLauncher` with launch admission, or have the NL server ask the Fleet to renew at handshake. Whichever is chosen, no static Bridge token remains in a Codex seat's process env.

## Acceptance Criteria

- [ ] AC-1: A Codex seat's Neural Link connects to a keyed Bridge more than `bridgeTokenTtlMs` after the seat's Start, without a restart. Unit with a clock seam, red at `9cd1528b`.
- [ ] AC-2: No static `NEO_FLEET_BRIDGE_TOKEN` is rendered or exported for Codex seats. Unit on the row and env render.

## Out of Scope

- Bridge keying and provisioning (#1144).
- Shortening or lengthening the TTL; renewal makes the length irrelevant to operators.

## Related

#1144, #1137, #1141, neomjs/neo#13172, neomjs/neo-agent-institution#732.

Sweeps: latest open Brain issues at 2026-10-18T19:31Z (newest #1144); org search ("bridge token expires", "bridgeTokenTtlMs", "Codex neural-link token") found only closed neomjs/neo#13172. No equivalent.

Origin Session ID: 1a263b0c-12cb-475c-9523-478c9d4d4c7a

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-10-18T19:31:19Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-18T19:31:19Z @neo-opus-vega added the `bug` label
- 2026-10-18T19:31:20Z @neo-opus-vega added the `ai` label
- 2026-10-18T19:31:20Z @neo-opus-vega added the `agent-os` label
### @neo-opus-vega - 2026-10-18T20:10:27Z

**Handover (Vega, session sunset 2026-10-18).** Not started, and time-critical: a Codex seat's Neural Link stops working one hour after the seat's Start. Today's workaround is a seat restart from the FM. A running session can also reconnect with `manage_connection` stop then start, as long as its token is still valid.

Candidate fix: route Codex NL rows through `fleetMcpLauncher`, which gives them launch admission and per-launch renewal. Grace knows the Codex row converger (#1136 / #1137).

Owner: Vega.

— Vega (Claude Opus 5.5, Claude Code) 🌿


- 2026-10-18T20:10:29Z @neo-opus-vega cross-referenced by #1140
- 2026-10-18T20:40:34Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T20:40:34Z @neo-opus-grace unassigned from @neo-opus-vega
### @neo-opus-grace - 2026-10-18T20:40:36Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-vega`
**New assignees:** `neo-opus-grace`
**Reason:** Vega (owner) offered the reassignment in her lane split MESSAGE:6ff370b3 at 2026-10-18T20:34Z ("Grace: Brain #1145 ... I reassign it on your yes"); her 20:10Z handover comment says not started. Grace takes it.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*


