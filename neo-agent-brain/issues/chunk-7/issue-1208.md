---
id: 1208
title: Fleet boot dies whenever it must start the Neural Link Bridge
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-19T16:42:58Z'
updatedAt: '2026-10-19T17:19:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1208'
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
closedAt: '2026-10-19T17:19:59Z'
---
# Fleet boot dies whenever it must start the Neural Link Bridge

## Context

A packaged rehearsal on Brain `dev` `883278aa` (#1193 merged) with Institution `9dc5cdf` failed at Fleet boot, in its own root and on its own NL port with nothing listening there:

```
[fleet] dev server failed to start: Missing Fleet dataDir: Neural Link Bridge stdio log requires a resolved Fleet data directory
```

The Fleet process exited with code 1 before its transport was ready.

## The Problem

`settleFleetBridge` (`ai/services/fleet/settleFleetBridge.mjs`, start branch) resolves the Bridge's stdio log path with `getFleetBridgeStdioLogPath({dataDir: registry?.dataDir ?? FleetRegistryService.dataDir, logFilePath})`.
- `dataDir` there is the registry's instance override, which production never sets. The resolved directory is `FleetRegistryService.getDataDir()`, the override or `AiConfig.fleet.dataDir`.
- The resolution sits before the start's `try`, so the throw rejects `settleFleetBridge`, and `devFleetServer`'s `boot()` exits.
- `FleetBridgeKeyProvisioning.spec` sets `FleetRegistryService.dataDir = <temp>`, so every arm saw a set override and CI stayed green.

When it fires: only on the start branch, when nothing answers on the NL port.
- **An outside operator's fresh install has no Bridge:** its FM's Fleet never boots.
- **The team machine** boots today because the host-edge Bridge answers on 8081. An FM start while that Bridge is down kills the Fleet too.

Second occurrence of one class: #1140 was `FleetLifecycleService` passing `this.registry` (only tests set it) where production needed `this.getRegistry()`. A test override field read where the resolver belongs passes every spec that sets the override.

Design authority: #1193's own contract for the start branch. Its JSDoc and the review's RA-3 disposition say a start failure becomes `bridge-unavailable`, never a dead Fleet. #1144's AC-1: a Bridge started on the product path.

## The Architectural Reality

- `FleetRegistryService.getDataDir()` is the resolver the Fleet package already reads through (`FleetTenantService`, `ai/scripts/fleet/forgeConnections.mjs`, `seatOperators.mjs`).
- `settleFleetBridge` receives the registry as `options.registry`, so the start branch can ask it.

## The Fix

1. The log path resolves through `registry.getDataDir()`.
2. The resolution moves inside the start's `try`, so any failure to place the log (path, permissions) reports `bridge-unavailable` with its reason, and Fleet boot continues.

## Acceptance Criteria

- [ ] AC-1 (red first): `settleFleetBridge` with a registry whose instance override is unset, but whose `getDataDir()` resolves, starts the Bridge and writes its log under that directory. Red at `883278aa`: it throws `Missing Fleet dataDir`.
- [ ] AC-2: a log path that cannot be opened resolves `bridge-unavailable` with its reason; `settleFleetBridge` never rejects for it.
- [ ] AC-3 (Post-Merge Validation): a packaged rehearsal on a candidate carrying the fix boots its Fleet, and the Fleet's line reports the Bridge `started`. Residual-Owner: neomjs/neo-agent-institution#752 (its AC-1 runs exactly this candidate).

## Out of Scope

- The smokes' Bridge port and teardown: neomjs/neo-agent-institution#752.
- The Bridge's owner: #1203.

## Avoided Traps

- **Reading `AiConfig.fleet.dataDir` directly:** it skips the registry's override, which tests and isolated callers rely on. The registry owns its directory.
- **Keeping the spec's `dataDir` override as the only arm:** that is what hid the defect. The red arm runs without it.

## Related

#1193 (introduced it) · #1144 · #1140 (same class) · #1203 · neomjs/neo-agent-institution#752 · neomjs/neo-agent-institution#732

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-19T16:42Z, no equivalent. A2A claim sweep: my claim to Julia and Vega at 16:42Z; no other claim on it. Memory Core sweep: Julia's 10-18 diagnosis of #1140 (the same override-vs-resolver class); no prior decision on the log path. Own-assignment sweep: #1204 (identity stamp) is unrelated.

Origin Session ID: 5370aea2-ad50-472b-92b7-b46d50dee583
Retrieval Hint: "settleFleetBridge Missing Fleet dataDir FleetRegistryService.dataDir getDataDir Fleet boot dies start branch"

## Timeline

- 2026-10-19T16:42:58Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-19T16:43:00Z @neo-opus-ada added the `bug` label
- 2026-10-19T16:43:00Z @neo-opus-ada added the `ai` label
- 2026-10-19T16:43:00Z @neo-opus-ada added the `agent-os` label
- 2026-10-19T16:45:26Z @neo-opus-ada cross-referenced by PR #1209
- 2026-10-19T17:19:59Z @tobiu referenced in commit `1f875a8` - "fix(fleet): the Bridge's log resolves the registry's data dir, and a log it cannot place never ends Fleet boot (#1208) (#1209)

settleFleetBridge placed the started Bridge's stdio log under registry.dataDir, the
registry's instance override that only tests set. In production the override is empty,
the path threw before the start's try, and devFleetServer's boot exited, so a Fleet
that had to start its Bridge never booted. The log now resolves through
registry.getDataDir(), inside the try, so a log that cannot be placed reports
bridge-unavailable with FLEET_DATA_DIR_MISSING and Fleet boot continues."
- 2026-10-19T17:19:59Z @tobiu closed this issue

