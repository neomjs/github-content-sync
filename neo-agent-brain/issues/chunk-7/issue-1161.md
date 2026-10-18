---
id: 1161
title: Warm-worker graph clears drop AgentIdentity boot seeds across specs
state: OPEN
labels:
  - bug
  - ai
  - testing
  - agent-os
assignees:
  - neo-gemini-pro
createdAt: '2026-10-18T22:44:46Z'
updatedAt: '2026-10-18T22:44:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1161'
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
# Warm-worker graph clears drop AgentIdentity boot seeds across specs

## Context

- **Design authority:** `ai/services/memory-core/GraphService.mjs:204-213` (`provisionMissingBootSeeds`), `ai/services/memory-core/lifecycle/SystemLifecycleService.mjs:35-53`, `test/playwright/unit/ai/services/memory-core/util.mjs:206-221` (`TestLifecycleHelper.cleanupGraphService`), `test/playwright/unit/ai/services/fleet/fleetMailboxMirrorAdapter.spec.mjs:422-448`
- **Live latest-open sweep:** checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T22:43:00Z` (`#1158`..`#1013`); no duplicate found.
- **A2A in-flight claim sweep:** checked latest 15 messages (`status: 'all'`) at `2026-10-18T22:43:17Z`; zero competing `[lane-claim]` or `[lane-intent]` on this surface (`@neo-gpt-sophie` confirmed in `MESSAGE:f1e1c7ab-23c1-46e1-b4c1-c3292878cb4e` that she has not started a competing fix).
- **Memory Core rationale & own-assignment sweep:** `query_raw_memories` (`GoldenPathSynthesizer OWNER_BENCHED_LANE provisionMissingBootSeeds cleanupGraphService fleetMailboxMirrorAdapter`) and `gh issue list --state open --assignee @me` (`0` open issues) confirmed no prior contradictory decision.
- **Promote defect-note:** Promotes `defect-note: util.mjs:208 cleanupGraphService('clear') wipes AgentIdentity seed nodes without resetting GraphService._initPromise, flaking GoldenPathSynthesizer.spec.mjs:3085 in CI (seen 22:10Z, 0c8aaeec & #1155)` (`MESSAGE:66527d78-3b59-4e86-97c7-8e35270fc9a7`), observed across CI runs on `#1152`, `#1154`, and `#1155`.

## The Problem

In `neomjs/neo-agent-brain` unit CI (`--workers=1` or multi-file worker batches), `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs:3085` (`synthesizeGoldenPath renders Work-Graph Stall Inference from issue sync and PR state`) intermittently fails with:

```text
Error: expect(received).toContain(expected) // indexOf
Expected substring: "OWNER_BENCHED_LANE"
```

Deterministic 2-file reproduction on `origin/dev`:

```bash
npx playwright test --config test/playwright/playwright.config.unit.mjs \
  test/playwright/unit/ai/services/fleet/fleetMailboxMirrorAdapter.spec.mjs \
  test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs \
  -g "granted: the row thread fact|synthesizeGoldenPath renders Work-Graph Stall Inference" \
  --workers=1
```

## The Architectural Reality

On cold worker boot, `GraphService.initAsync()` (`ai/services/memory-core/GraphService.mjs:174-226`) mounts `this.db` and calls `this.provisionMissingBootSeeds()` (`:209`), seeding the canonical `AgentIdentity` and `BroadcastSentinel` roots from `ai/graph/bootSeedManifest.mjs`.

When an earlier spec in the same worker clears the shared `GraphService.db` without destroying it, two gaps leave the worker graph without canonical boot seeds for subsequent specs:

1. **`fleetMailboxMirrorAdapter.spec.mjs:422-448` wipes `GraphService.db` in `beforeEach` and leaves its 4 fake identities in `afterAll`:**
   - `beforeEach` (`:431-440`) runs `GraphService.db.nodes.clear()`, `GraphService.db.edges.clear()`, `GraphService.db.vicinityLoadedNodes.clear()`, `await GraphService.db.storage.clear()`, and `DELETE FROM GraphLog`, then inserts `@subject-agent`, `@viewer-agent`, `@outsider-agent`, `@sender-agent`, `AGENT:*`, and `THREAD:15269`.
   - `afterAll` (`:422-427`) only calls `cleanupChromaManager()` and restores `GraphService.db.autoSave`, leaving its test nodes in `GraphService.db` and all canonical `AgentIdentity` boot seeds (`@neo-gemini-pro`, `@neo-gpt`, etc.) absent.
2. **`TestLifecycleHelper.cleanupGraphService(..., 'clear')` (`test/playwright/unit/ai/services/memory-core/util.mjs:208-221`) and `SystemLifecycleService.initAsync()` (`ai/services/memory-core/lifecycle/SystemLifecycleService.mjs:35-53`) do not restore canonical boot seeds when `GraphService.db` remains mounted:**
   - `cleanupGraphService(..., 'clear')` clears `GraphService.db.nodes`, `GraphService.db.edges`, `GraphService.db.vicinityLoadedNodes`, and `GraphService.db.storage.clear()`, and calls `resetMemoryCoreLifecycle()`, which nulls `SystemLifecycleService._initPromise` (`util.mjs:44`) while leaving `GraphService.db` non-null and `GraphService._initPromise` resolved.
   - When the next spec in the worker (`GoldenPathSynthesizer.spec.mjs:106`) calls `await SystemLifecycleService.initAsync()`, `SystemLifecycleService.initAsync()` only awaits `GraphService.ready()`. Because `GraphService.initAsync()` guards at `:177` with `if (this.db || this._initPromise) return`, `GraphService.provisionMissingBootSeeds()` never runs again on warm-worker lifecycle boots.

## The Fix

1. **`ai/services/memory-core/lifecycle/SystemLifecycleService.mjs`:**
   - After `await GraphService.ready()`, if `GraphService.db?.storage?.db` is mounted, call `GraphService.provisionMissingBootSeeds()` (inside the same non-fatal `try / catch` guard used in `GraphService.initAsync()`) so that every `SystemLifecycleService.initAsync()` boot after `resetMemoryCoreLifecycle()` guarantees the canonical boot-seed manifest is present, matching cold-worker boot.
2. **`test/playwright/unit/ai/services/memory-core/util.mjs` (`TestLifecycleHelper.cleanupGraphService`):**
   - In the `strategy === 'clear'` branch (`:208-221`), also reset `GraphLog` (`DELETE FROM GraphLog`) and `GraphService.db.lastSyncId = 0` when `GraphService.db.storage?.db` is present, and re-provision `GraphService.provisionMissingBootSeeds()` so clearing a spec's graph returns the worker's mounted `GraphService.db` to its freshly-booted baseline.
3. **`test/playwright/unit/ai/services/fleet/fleetMailboxMirrorAdapter.spec.mjs`:**
   - In `test.afterAll` (`:422-427`), call `await TestLifecycleHelper.cleanupGraphService(GraphService, LifecycleService, null, null, 'clear')` so the spec does not leak its fixture nodes (`@subject-agent`, `@viewer-agent`, `@outsider-agent`, `@sender-agent`, `THREAD:15269`) or leave the worker graph unseeded.
4. **Unit test coverage:**
   - Add a dedicated unit test verifying that after `GraphService.db` and `GraphService.db.storage` are cleared and `resetMemoryCoreLifecycle()` runs, both `TestLifecycleHelper.cleanupGraphService(..., 'clear')` and `SystemLifecycleService.initAsync()` restore the canonical `AgentIdentity` boot seeds.

## Decision Record impact

`none` — preserves the existing `GraphService.provisionMissingBootSeeds()` additive boot-seed contract across warm-worker test lifecycle boundaries.

## Acceptance Criteria

- [ ] `TestLifecycleHelper.cleanupGraphService(GraphService, SystemLifecycleService, ..., 'clear')` clears `GraphLog` / `lastSyncId` and restores canonical boot seeds via `GraphService.provisionMissingBootSeeds()` when `GraphService.db?.storage?.db` is mounted.
- [ ] `SystemLifecycleService.initAsync()` calls `GraphService.provisionMissingBootSeeds()` when `GraphService.db?.storage?.db` is mounted after `await GraphService.ready()`.
- [ ] `fleetMailboxMirrorAdapter.spec.mjs` cleans up `GraphService` via `TestLifecycleHelper.cleanupGraphService(..., 'clear')` in `afterAll`.
- [ ] The 2-file `--workers=1` reproducer (`fleetMailboxMirrorAdapter.spec.mjs` followed by `GoldenPathSynthesizer.spec.mjs:3085`) passes deterministically.

## Out of Scope

- Changing `GraphService.initAsync()`'s singleton guard (`if (this.db || this._initPromise) return`).
- Changing `GoldenPathSynthesizer`'s `OWNER_BENCHED_LANE` stall inference rules.

## Related

- `neomjs/neo-agent-brain#1158` / `neomjs/neo-agent-brain#1159` (sibling cross-file worker leak fix in `FleetLifecycleService.spec.mjs`)
- `neomjs/neo-agent-brain#1152`, `neomjs/neo-agent-brain#1154`, `neomjs/neo-agent-brain#1155` (PRs whose CI runs hit this flake)

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb
Retrieval Hint: "GoldenPathSynthesizer OWNER_BENCHED_LANE provisionMissingBootSeeds cleanupGraphService fleetMailboxMirrorAdapter"


## Timeline

- 2026-10-18T22:44:47Z @neo-gemini-pro added the `bug` label
- 2026-10-18T22:44:47Z @neo-gemini-pro added the `ai` label
- 2026-10-18T22:44:47Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-18T22:44:47Z @neo-gemini-pro added the `testing` label
- 2026-10-18T22:44:47Z @neo-gemini-pro added the `agent-os` label
- 2026-10-18T22:51:48Z @neo-gemini-pro cross-referenced by PR #1163
- 2026-10-18T23:07:12Z @neo-opus-ada cross-referenced by #1164

