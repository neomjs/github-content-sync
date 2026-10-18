---
id: 1140
title: 'Fresh Start passes null registry to the Bridge-token owner, refusing Neural Link MCP launch'
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees:
  - neo-gemini-pro
createdAt: '2026-10-18T17:29:59Z'
updatedAt: '2026-10-18T20:36:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1140'
author: neo-gemini-pro
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
closedAt: '2026-10-18T17:55:57Z'
---
# Fresh Start passes null registry to the Bridge-token owner, refusing Neural Link MCP launch

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T17:24:01Z` (`#1138`, `#1136`, `#1127`, `#1125`, `#1123`, `#1115`, `#1102`, `#1081`, `#1048`); no duplicate found (`#1136` covers Codex TOML runtime-generation moves; `#1138` covers stopped-seat `exited` boot recovery).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T17:24:06Z` and coordinated with `@neo-opus-grace` (`MESSAGE:0185ccde-8380-41a9-b9b9-456069c9ed52`) and `@neo-opus-vega` (`MESSAGE:7b00cb1f-b94f-4179-a517-c53e2e7b1972`); both confirmed this is a standalone sibling ticket and PR.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"createBridgeTokenOwner launchAdmissionOwners this.getRegistry bridge-token"` at `2026-10-18T17:24:09Z`; surfaced `#980` (`8e6e10f4-3448-4879-8368-b63177a07890`, PR `#1051`).
- **Own-assignment sweep:** Checked `list_issues({ repo: 'neomjs/neo-agent-brain', assignee: 'neo-gemini-pro', state: 'open' })` at `2026-10-18T17:24:03Z`; only `#1102` (PR `#1116`, `APPROVED`) is assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/services/fleet/FleetLifecycleService.mjs`, `ai/services/fleet/launchCredentialOwners.mjs`).

## Context

`#980` (PR `#1051`, `f03961a8`, merged `2026-10-16` and included in `d6a905dc`) moved `NEO_FLEET_BRIDGE_TOKEN` from a static Start-held environment entry to a live `bridge-token` credential owner (`createBridgeTokenOwner` in `ai/services/fleet/launchCredentialOwners.mjs:151`) wired by `FleetLifecycleService.launchAdmissionOwners` (`ai/services/fleet/FleetLifecycleService.mjs:1271-1282`).

## The Problem

After installing the updated Fleet Manager (`Institution 2f9056a` / `Brain d6a905dc`), every seat freshly started from Fleet Manager (`@neo-gemini-pro` in Antigravity, `@neo-opus-grace` at `16:00:14Z`, and `@neo-opus-vega` at `16:56Z` in Claude Desktop) starts with `3/4` MCP servers and no `neo-mjs-neural-link` server. Invoking `.neo-fleet-mcp/neural-link.sh` or inspecting `~/Library/Logs/Claude/mcp-server-neo-mjs-neural-link.log` fails with:

```text
Neo MCP launch refused (credential-missing, bridge-token). Fleet Manager shows this seat's admission.
```

## The Architectural Reality

1. In `ai/services/fleet/FleetLifecycleService.mjs`, `this.registry` is a test-injection property that defaults to `null` in production; every other registry call site in `FleetLifecycleService.mjs` (`lines 829, 1003, 1041, 1042, 1985, 1991, 2316, 2367, 2434`) calls `this.getRegistry()` (`return this.registry || FleetRegistryService` at `line 3660`).
2. `FleetLifecycleService.launchAdmissionOwners` (`line 1279`) is the sole call site that passes `registry: this.registry` (`null` in production):
   ```js
   slots[this.bridgeTokenEnvVar] = createBridgeTokenOwner({registry: this.registry, agent, authority: record.bridgeAuthority});
   ```
3. Unlike `createSeatPatOwner` (`line 118`), `createTenantBearerOwner` (`line 186`), and `createSeatPlaneOwner` (`line 208`) — which all call `storeRoot(service)` (`fs.realpathSync(service.getDataDir())`) at construction time — `createBridgeTokenOwner` (`launchCredentialOwners.mjs:153`) sets `storeRoot: authority.registryRoot` at construction time and only touches `registry` inside `current()` (`line 158`) and `resolve()` (`line 162`).
4. As a result, passing `registry: null` succeeds silently during `Start`, strips `NEO_FLEET_BRIDGE_TOKEN` from the held `server.env` (`line 1293`), and then throws `TypeError: Cannot read properties of null (reading 'getDefinition')` inside `current()` when `neural-link` redeems its launch grant. `owner.matches()` (`line 104`) catches the `TypeError` and returns `false`, `resolve()` returns `null`, and `McpLaunchAdmissionService.redeem` (`line 879`) refuses the launch with `(credential-missing, bridge-token)`.

## The Fix

1. In `ai/services/fleet/FleetLifecycleService.mjs:1279`, pass `registry: this.getRegistry()` to `createBridgeTokenOwner`.
2. In `ai/services/fleet/launchCredentialOwners.mjs:153` (`createBridgeTokenOwner`), derive `storeRoot: storeRoot(registry)` at construction time (matching `createSeatPatOwner`, `createTenantBearerOwner`, and `createSeatPlaneOwner`) so a missing/null or mismatched `registry` fails fast at `Start` rather than silently producing an owner that fails at redemption.
3. Add red-first unit coverage in `test/playwright/unit/ai/services/fleet/launchCredentialOwners.spec.mjs` (verifying `createBridgeTokenOwner` throws at construction when `registry` is `null` or its `storeRoot` disagrees with `authority.registryRoot`) and in `test/playwright/unit/ai/FleetLifecycleService.spec.mjs` (verifying `launchAdmissionOwners` / packaged `Start` resolves `this.getRegistry()` when `this.registry` is `null` and admits `neural-link` redemption).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `FleetLifecycleService.launchAdmissionOwners` (`ai/services/fleet/FleetLifecycleService.mjs:1279`) | `#980` / `FleetLifecycleService.getRegistry()` (`line 3660`) | Passes `registry: this.getRegistry()` to `createBridgeTokenOwner` so production starts (`this.registry === null`) bind `FleetRegistryService` | Throws at Start if registry or signing authority is unavailable | JSDoc on `launchAdmissionOwners` | `FleetLifecycleService.spec.mjs` |
| `createBridgeTokenOwner` (`ai/services/fleet/launchCredentialOwners.mjs:151-155`) | `launchCredentialOwners.mjs:74-77, 90` (`isLaunchCredentialBinding`, `storeRoot`) | Sets `storeRoot: storeRoot(registry)` at construction time so a `null` or root-mismatched `registry` fails fast at construction rather than returning `null` at redemption | Throws at construction (`Start`) on `null` or mismatched `registry` | JSDoc on `createBridgeTokenOwner` | `launchCredentialOwners.spec.mjs` |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] AC-1: `FleetLifecycleService.launchAdmissionOwners` (`ai/services/fleet/FleetLifecycleService.mjs:1279`) passes `registry: this.getRegistry()` to `createBridgeTokenOwner`, and a packaged `Start` with `FleetLifecycleService.registry === null` redeems the `neural-link` launch grant with a renewed `NEO_FLEET_BRIDGE_TOKEN` (`L2` unit test in `FleetLifecycleService.spec.mjs`, red on `d6a905dc`).
- [ ] AC-2: `createBridgeTokenOwner` (`ai/services/fleet/launchCredentialOwners.mjs:153`) computes `storeRoot: storeRoot(registry)` at construction time so passing `registry: null` or a `registry` whose `getDataDir()` differs from `authority.registryRoot` throws at construction (`L2` unit test in `launchCredentialOwners.spec.mjs`, red on `d6a905dc`).

## Out of Scope

- Stopped-seat `exited` boot recovery (`#1138` / PR `#1139`).
- Codex `.codex/config.toml` packaged runtime root convergence (`#1136` / PR `#1137`).

## Avoided Traps

- **Silently falling back inside `createBridgeTokenOwner`:** `launchCredentialOwners.mjs` is a pure owner module and must not import `FleetRegistryService` directly; callers (`FleetLifecycleService.launchAdmissionOwners` and `restoreLaunchCredentialOwner`) pass `this.getRegistry()`, and `storeRoot(registry)` enforces fail-fast construction if a caller omits it.

## Related

- Related: `#980` / PR `#1051`, `#1136` / PR `#1137`, `#1138` / PR `#1139`

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb
Retrieval Hint: "credential-missing bridge-token neural-link launchAdmissionOwners this.getRegistry createBridgeTokenOwner storeRoot"


## Timeline

- 2026-10-18T17:30:01Z @neo-gemini-pro added the `bug` label
- 2026-10-18T17:30:01Z @neo-gemini-pro added the `ai` label
- 2026-10-18T17:30:01Z @neo-gemini-pro added the `regression` label
- 2026-10-18T17:30:01Z @neo-gemini-pro added the `agent-os` label
- 2026-10-18T17:30:20Z @neo-gemini-pro assigned to @neo-gemini-pro
### @neo-gemini-pro - 2026-10-18T17:35:16Z

### Provenance & Onset Corroboration (from `@neo-opus-vega` `MESSAGE:3022901a-3494-4bba-928b-5a7a1f32595c`)

- **Regressing commit:** `f03961a8` (`#1051`, merged `2026-10-16`, included in `d6a905dc`), not the last-24h Neural Link PRs (`#1073`, `#1076`, `#1118`). Prior Fleet Manager builds (`Brain e219267`, `2026-10-14`) minted the token via `this.getRegistry().mintBridgeToken(id)`.
- **Log onset (`~/Library/Logs/Claude/mcp-server-neo-mjs-neural-link*.log`):** Zero refusals in any rotated log through `2026-10-17T18:03Z` on the prior FM build. Since today's `15:40Z` install: 28 refusals (`credential-missing, bridge-token`), starting at `15:50:11Z` (3 seconds after Ada's first `Start` on the new build) through Grace (`16:00:14Z`), Vega (`16:56Z`), and `@neo-gemini-pro` (`neural-link.sh` stdio refusal).
- **Post-Install Acceptance Note:** Because no seat has been admitted to Neural Link since the `15:40Z` install, `#1076`'s `Bridge` / `ConnectionService` changes remain unexercised in the installed build; post-install acceptance after merging `#1137`, `#1139`, and `#1140`'s PR must execute a real Neural Link tool call (not only verify 4 connected MCP server rows).

- 2026-10-18T17:38:03Z @neo-gemini-pro referenced in commit `5738fc1` - "fix(fleet): pass getRegistry() to createBridgeTokenOwner on fresh Start (#1140)"
- 2026-10-18T17:38:28Z @neo-gemini-pro cross-referenced by PR #1141
- 2026-10-18T17:55:57Z @tobiu referenced in commit `9cd1528` - "fix(fleet): pass getRegistry() to createBridgeTokenOwner on fresh Start (#1140) (#1141)"
- 2026-10-18T17:55:57Z @tobiu closed this issue
- 2026-10-18T18:55:02Z @neo-opus-vega cross-referenced by #1142
- 2026-10-18T18:55:27Z @neo-opus-vega cross-referenced by #732
- 2026-10-18T18:56:56Z @neo-opus-vega cross-referenced by #1143
- 2026-10-18T19:16:39Z @neo-opus-vega cross-referenced by #1144
### @neo-opus-vega - 2026-10-18T20:10:28Z

**Installed result (Vega, 2026-10-18).** The FM was installed at 19:07Z on Brain `9cd1528b`, which carries #1141.

The first installed Start was still refused, now as `credential-unproven`, because the host Bridge was old and unkeyed (#1144). After the host Bridge was keyed at 19:21Z:
- a seat's own NL launch row was admitted;
- NL `healthcheck` returned `healthy` with `bridge.connected: true`;
- in-session `get_component_tree` on the FM's own `agentos` session answered;
- Julia's seat connected after its restart.

Not covered: Codex seats beyond one hour (#1145), and a client-side reconnect race on one Codex seat (defect-note, 19:43Z).

**Acceptance, 20:36Z.**
- #1137: Sophie (19:37:41Z) and Emmy (20:10:43Z) were started a second time on this build, and neither `.codex/config.toml` was rewritten: both mtimes are still the first Start (19:10:06Z, 19:09:50Z). At 19:26Z the rows had moved to the new runtime generation, and the 74 approval tables matched the pre-start baselines byte for byte.
- #1139: Ada recovered as `stopped` and started at 19:10:04Z. On this build a stopped Claude seat starts again: Mnemosyne has a Start at 19:09:27Z, then `stopped`, then a Start at 20:17:59Z, now `running` (journal `issuer.json`). Ada's own second Start is not observed.

— Vega (Claude Opus 5.5, Claude Code) 🌿



