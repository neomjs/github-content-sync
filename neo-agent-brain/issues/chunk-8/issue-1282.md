---
id: 1282
title: The Neural Link runs standalone from its own npm package
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-25T23:56:54Z'
updatedAt: '2026-10-26T01:30:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1282'
author: neo-opus-grace
commentsCount: 0
parentIssue: 20114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 46 neo-app wires the Neural Link into the workspace''s MCP configs'
closedAt: '2026-10-26T01:30:18Z'
---
# The Neural Link runs standalone from its own npm package

## Context

This is the Brain leaf of neomjs/neo#20114, graduated from neomjs/neo D#20113 (N1). The Neural Link is how an outside agent verifies the app it builds. Today the server runs only from a Brain checkout.

## The Problem

`neo-agent-brain` is `private` and not on npm. The Neural Link server imports Brain modules and an untracked `./config.mjs` overlay. Packed as it is, its plane would default to wherever the code sits, and nothing refuses an npx cache. So its logs would quietly land in `~/.npm/_npx/…`.

## The Architectural Reality

- `ai/mcp/server/neural-link/{mcp-server,run-bridge,Server,configBase}.mjs`. `Server` does not override `isPlaneMember()`, so `assertPlaneIdentity()` returns early (`ai/mcp/server/BaseServer.mjs:634-689`). `PLANE_MEMBER_PATHS` is `['logPath']` (`configBase.mjs:182`).
- `ai/scripts/setup/initServerConfigs.mjs:40` resolves `neo.mjs/package.json` at import. The overlays `ai/config.mjs` and `ai/mcp/server/neural-link/config.mjs` are untracked and generated at `prepare`, which a registry install never runs.
- `ai/planeConfig.mjs:161`: `assertPlaneDataRootPlacement` refuses `.app` paths, and unwritable ones on request, and nothing else.
- `ai/services/neural-link/ConnectionService.mjs`: `spawnBridge` runs `npm run ai:server-neural-link` inside `--cwd`. It refuses an unresolved `cwd` and a `cwd` without that script (`getCwdFinding`). With a Fleet bridge token it dials the host-edge Bridge and spawns nothing.
- `ai/mcp/server/neural-link/diagnosticPathAttestation.mjs:95` returns early without a launcher commitment. So `memoryCoreDbPathProd` needs no standalone value.
- `ai/services.host.mjs` already separates the Neural Link services from the store-backed ones.
- ADR-0019 §10.4 (placement guard) and §10.7 (the deployment-profile matrix). The implementer reads ADR-0019 in full first (the AiConfig gate).

## The Fix

1. **ADR-0019 §10.7:** a standalone row, in or before this PR:
   - the package launcher places the plane at `~/.neo-ai/neural-link` by env before start;
   - `logPath` is placed beneath it, and `memoryCoreDbPathProd` keeps its derived default;
   - there is no wake lane and no host publication besides the Bridge's loopback port;
   - the amendment is checked against §5 (ADR §6.4).
2. **A pack step**, placed by structural pre-flight (the sibling is `ai/scripts/setup`). It assembles the server's import closure and materializes both config overlays. It writes the package manifest:
   - `mcpName: io.github.neomjs/neural-link` and a launcher `bin`;
   - `scripts['ai:server-neural-link']` pointing at the packaged `run-bridge.mjs`;
   - dependencies on `neo.mjs` and the closure's runtime packages.
3. **The launcher** sets the standalone placement env, then starts `mcp-server.mjs` with the package root as `--cwd`.
4. **`ai/mcp/server/neural-link/server.json`** gets its `packages` entry (`registryType: npm`).
5. **No change** to `ConnectionService`, `BaseServer` or `planeConfig`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| the Neural Link npm package (name per the operator, OQ5) | [MCP Registry npm package type](https://modelcontextprotocol.io/registry/package-types) | runs the Neural Link MCP server with no Brain checkout | none | the package README | cold `npx` receipt |
| the launcher's placement env | ADR-0019 §10.7 standalone row | relocates the plane before the server starts | none: no inference from where the code lives | the ADR row | unit spec on the env the launcher passes |
| `neural-link/server.json` `packages` | registry `server.json` schema | npm identifier and version | none | none | `McpRegistryManifests.spec.mjs` |

## Acceptance Criteria

- [ ] ADR-0019 §10.7 carries the standalone row, checked against §5.
- [ ] The packed tarball runs on a clean HOME with no Brain checkout:
  - an `npx` launch answers the MCP `healthcheck`;
  - the Bridge auto-spawns from the package root;
  - the logs land under `~/.neo-ai/neural-link/logs`.
- [ ] With a Fleet bridge token set, it dials the host-edge Bridge and spawns nothing, as today.
- [ ] `ConnectionService`, `BaseServer` and `planeConfig` are unchanged.
- [ ] `neural-link/server.json` declares the npm package, and `McpRegistryManifests.spec.mjs` stays green.
- [ ] Post-merge (the operator): the first npm publish and its trusted publisher.
- [ ] Post-merge (the operator): the registry publish, per `neo-agent-brain#1265`'s post-merge AC.

## Out of Scope

- The other five servers' packages and registry entries.
- A hosted Knowledge Base.
- The scaffold wiring (its own leaf under neomjs/neo#20114).

## Related

Refs neomjs/neo#20114 · graduated from neomjs/neo D#20113 · `#1265` (the registry manifests) · `#481` (the host-edge Bridge, unchanged).

Sweeps: live latest-open (20, `2026-10-25T23:52:52Z`), A2A (30, `23:53Z`), MC (1 problem-noun query) and own-assignment (2 open, `#684` and `#54`, none overlapping) found no equivalent. Structure map: `origin/dev` `0d8cac84`.

Origin Session ID: 5597537c-fa0a-4b88-8847-edddb7492a11

Retrieval Hint: `query_raw_memories("Neural Link standalone npm package launcher plane placement ADR-0019 10.7 npx")`


## Timeline

- 2026-10-25T23:56:55Z @neo-opus-grace added the `enhancement` label
- 2026-10-25T23:56:55Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-25T23:56:55Z @neo-opus-grace added the `ai` label
- 2026-10-25T23:56:55Z @neo-opus-grace added the `architecture` label
- 2026-10-25T23:56:55Z @neo-opus-grace added the `agent-os` label
- 2026-10-25T23:57:14Z @neo-opus-grace added parent issue #20114
- 2026-10-25T23:57:16Z @neo-opus-grace marked this issue as blocking #46
- 2026-10-25T23:58:11Z @neo-opus-grace cross-referenced by #1265
- 2026-10-26T00:50:01Z @neo-opus-grace cross-referenced by PR #1283
- 2026-10-26T01:30:18Z @tobiu referenced in commit `da24eba` - "feat(neural-link): the Neural Link runs standalone from its own npm package (#1282) (#1283)

An outside developer's MCP client can now run the Neural Link with no Brain
checkout: `npm run pack:neural-link -- --out <dir>` assembles the server's
import closure (acorn-walked from its three entrypoints), its runtime assets
and the config overlays `prepare` would write on a clean clone, plus a
manifest whose identity comes from one declaration, `server.json`: its name
is the `mcpName`, its npm `packages` entry the package name and version.
Nine dependencies, none native and none store-backed.

The package's bin is `ai/mcp/server/neural-link/standalone.mjs`. It declares
the standalone placement (`~/.neo-ai/neural-link`, `logPath` beneath it) by
env before the server starts, and passes the package root as `--cwd`, so
`ConnectionService#spawnBridge` starts the packaged Bridge through its
existing path with both refusals intact. No change to ConnectionService,
BaseServer or planeConfig. ADR-0019 §10.7 gains the standalone row.

Cold run from the packed tarball (fresh HOME, fresh npm cache): npx connects
in 2.9 s, 60 tools, healthcheck healthy with the Bridge spawned and connected,
logs under ~/.neo-ai/neural-link/logs."
- 2026-10-26T01:30:18Z @tobiu closed this issue

