---
id: 1090
title: Complete TextEmbeddingService snapshot seams and convert B4 specs
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - testing
  - architecture
assignees:
  - neo-gemini-julia
createdAt: '2026-10-17T22:38:29Z'
updatedAt: '2026-10-18T01:23:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1090'
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
closedAt: '2026-10-18T01:23:52Z'
---
# Complete TextEmbeddingService snapshot seams and convert B4 specs

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-17T22:37:00Z`; no duplicate found (`#541` closed via PR `#1086`).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-17T22:37:10Z`; no competing `[lane-claim]` or `[lane-intent]` on B4 shared-config write isolation.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"shared-config write findSharedConfigMutations TextEmbeddingService"`; surfaced `#541` and PR `#1086` Round-1 review (`5560361878`).
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me`; `#1090` self-assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/services/shared/serviceProxy.mjs`, `ai/services/memory-core/TextEmbeddingService.mjs`, `ai/services/knowledge-base/VectorService.mjs`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## The Problem

`#541` (resolved by PR `#1086`) established the `configSnapshot` / `createConfigSnapshot()` injection pattern on `TextEmbeddingService` and converted the largest single concentration of ADR-0019 B4 shared-config writes (`test/playwright/unit/ai/services/memory-core/TextEmbeddingService.retry.spec.mjs`, `80 -> 0` writes).

After `#1086`:
1. `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` reported `577 shared-config write(s) across 142 leaf/leaves` across `55` test files (`517` assignment, `60` `Object.assign`), including `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs` (`20` writes) and `test/playwright/unit/ai/services/knowledge-base/VectorService.batchFailureIsolation.spec.mjs` (`12` writes).
2. During Round-1 review of `#1086` (`5560361878`), `@neo-opus-ada` identified three concrete hardening items on `TextEmbeddingService`'s snapshot seam itself:
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.batchFailureIsolation.spec.mjs` (`:164, :167, :223`) is the sole external consumer of `TextEmbeddingService.openAiCompatibleHostFn_`; moving it onto `configSnapshot` allows deleting `openAiCompatibleHostFn_` entirely.
   - `set configSnapshot(value)` does not reset `this.ollamaProvider = null`, so a native-Ollama test that sets `configSnapshot` after `#getOllamaProvider()` has already cached a `Neo.ai.provider.Ollama` instance would keep the prior host/model.
   - `createConfigSnapshot()` copies only enumerated leaves rather than cloning the whole consumed subtrees (`localModels.embedding`, `ollama`, `openAiCompatible`, `orchestrator.lms`, `orchestrator.providerReadiness`), which could let a future `TextEmbeddingService` read silently fall back to destructuring defaults (`unloadRetryCount = 3`, etc.) under snapshot mode.

## The Architectural Reality

- `ai/services/memory-core/TextEmbeddingService.mjs` owns `configSnapshot`, `beforeSetConfigSnapshot(value)`, and `createConfigSnapshot()`. At the Brain's current `neo.mjs` pin (`7f16355a`, 2026-09-25, which predates `neomjs/neo#19269` / `a87a89e2`), `{...aiConfig.ollama}` on a Tier-2 child `ConfigProvider` only enumerates child-owned keys; walking `current.getParent()` is required until the pin advances past `a87a89e2`.
- `test/playwright/unit/ai/services/knowledge-base/VectorService.batchFailureIsolation.spec.mjs` accesses singletons through `SDK` (`ai/services.mjs`), which wraps singletons via `makeSafe` (`ai/services/shared/serviceProxy.mjs`). `makeSafe` traps `get` with `Reflect.get(target, prop, target)` but lacks a matching `set` trap, causing private-field (`#configSnapshot`) writes in setters to throw when invoked through `SDK.*`.
- `ai/services/knowledge-base/VectorService.mjs` `embedChunks` reads `{backoffBaseMs, batchSize, batchDelay, maxRetries, undeliverableTimeoutStrikes}` from `aiConfig`, which `VectorService.batchFailureIsolation.spec.mjs` mutates (`12` shared-config writes).

## The Fix

1. In `ai/services/shared/serviceProxy.mjs`:
   - Add `set(target, prop, value) { return Reflect.set(target, prop, value, target); }` to `makeSafe`'s `Proxy` handler so setters that write `#configSnapshot` work through `SDK.*`.
2. In `ai/services/shared/configSnapshot.mjs`:
   - Extract the shared snapshot seam (`isPlainSnapshotObject`, `cloneConfigSubtree`, and `validateConfigSnapshot`) so `TextEmbeddingService`, `VectorService`, and `#1091` consumers share one validator and subtree cloner (with `cloneConfigSubtree` JSDoc scoping the `getParent()` walk to the `7f16355a` pin and naming `a87a89e2` as its retirement trigger).
3. In `ai/services/memory-core/TextEmbeddingService.mjs` and `ai/services/knowledge-base/VectorService.mjs`:
   - Keep `#configSnapshot = null` on each instance, reset `this.ollamaProvider = null` on `TextEmbeddingService` snapshot transitions, retire `openAiCompatibleHostFn_`, and isolate `VectorService.embedChunks`'s 5 batch/retry leaves.
4. Convert `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs` (`20 -> 0` writes) and `test/playwright/unit/ai/services/knowledge-base/VectorService.batchFailureIsolation.spec.mjs` (`12 -> 0` writes) onto `createConfigSnapshot()`, and track the remaining `545` non-`TextEmbeddingService` writes across `53` files in `#1091`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `makeSafe` (`ai/services/shared/serviceProxy.mjs`) | ADR-0019 §4 B4 + `#1090` | Traps `set` with `Reflect.set(target, prop, value, target)` so `#private` instance fields work in setters called via `SDK.*` | Standard `Reflect.set` failure semantics | JSDoc on `serviceProxy.mjs` | Unit tests in `VectorService.batchFailureIsolation.spec.mjs` and `TextEmbeddingService.spec.mjs` |
| `isPlainSnapshotObject`, `cloneConfigSubtree`, `validateConfigSnapshot` (`ai/services/shared/configSnapshot.mjs`) | ADR-0019 §4 B4 + `#1090` | Shared recursive plain-object/array validator and parent-chain subtree cloner (retirement trigger: advancing `neo.mjs` pin past `a87a89e2`) | Throw `TypeError` on non-isolated inputs or nested proxies | JSDoc on `configSnapshot.mjs` | Unit tests in `TextEmbeddingService.spec.mjs`, `TextEmbeddingService.retry.spec.mjs`, `VectorService.batchFailureIsolation.spec.mjs` |
| `TextEmbeddingService.configSnapshot` / `createConfigSnapshot()` (`ai/services/memory-core/TextEmbeddingService.mjs`) | ADR-0019 §4 B4 + `#541` | Clones whole resolved subtrees via `cloneConfigSubtree`, resets `ollamaProvider` on snapshot change, and retires `openAiCompatibleHostFn_` | Throw `TypeError` via `beforeSetConfigSnapshot` | JSDoc on `TextEmbeddingService.mjs` | Unit tests in `TextEmbeddingService.spec.mjs`, `TextEmbeddingService.retry.spec.mjs`, `VectorService.batchFailureIsolation.spec.mjs` |
| `VectorService.configSnapshot` / `createConfigSnapshot()` (`ai/services/knowledge-base/VectorService.mjs`) | ADR-0019 §4 B4 + `#1090` | Isolates `embedChunks` batch/retry leaves (`backoffBaseMs`, `batchDelay`, `batchSize`, `maxRetries`, `undeliverableTimeoutStrikes`) for unit tests | Throw `TypeError` via `beforeSetConfigSnapshot` | JSDoc on `VectorService.mjs` | Unit tests in `VectorService.batchFailureIsolation.spec.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] `createConfigSnapshot()` on `TextEmbeddingService` copies full resolved plain-object subtrees for `localModels.embedding`, `ollama`, `openAiCompatible`, `orchestrator.lms`, and `orchestrator.providerReadiness`.
- [ ] `set configSnapshot(value)` resets `this.ollamaProvider = null` when the snapshot changes (verified by a unit test on the native Ollama path).
- [ ] `VectorService.batchFailureIsolation.spec.mjs` (`12 -> 0` writes) and `TextEmbeddingService.spec.mjs` (`20 -> 0` writes) use `configSnapshot` (`0` shared-config writes in `findSharedConfigMutations`), and `openAiCompatibleHostFn_` is removed from `TextEmbeddingService.mjs`.
- [ ] The remaining non-`TextEmbeddingService` B4 shared-config write concentrations (`545` writes across `53` test files) are decomposed and tracked in `#1091`.

## Out of Scope

- Converting the remaining `53` non-`TextEmbeddingService` test files (`545` writes), which are tracked in `#1091`.

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`; tests must inject an isolated `configSnapshot` rather than proliferating one function config per leaf.
- **Per-service module-scoped snapshot state or duplicated validators:** Fixed at the owner via `makeSafe`'s `set` trap (`ai/services/shared/serviceProxy.mjs`) and shared `ai/services/shared/configSnapshot.mjs`.

## Related

- Succeeds: `#541` / PR `#1086`
- Succeeded by: `#1091`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "TextEmbeddingService configSnapshot openAiCompatibleHostFn findSharedConfigMutations B4 VectorService serviceProxy"


## Timeline

- 2026-10-17T22:38:30Z @neo-gemini-julia added the `enhancement` label
- 2026-10-17T22:38:30Z @neo-gemini-julia added the `ai` label
- 2026-10-17T22:38:31Z @neo-gemini-julia added the `refactoring` label
- 2026-10-17T22:38:31Z @neo-gemini-julia added the `testing` label
- 2026-10-17T22:38:31Z @neo-gemini-julia added the `architecture` label
- 2026-10-17T22:38:43Z @neo-gemini-julia cross-referenced by #541
- 2026-10-17T22:40:03Z @neo-gemini-julia cross-referenced by PR #1086
- 2026-10-17T23:28:54Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-18T00:04:07Z @neo-gemini-julia cross-referenced by #1091
- 2026-10-18T00:06:08Z @neo-gemini-julia cross-referenced by PR #1092
- 2026-10-18T00:25:29Z @neo-gemini-julia changed title from **Complete TextEmbeddingService snapshot seams and remaining B4 test slices** to **Complete TextEmbeddingService snapshot seams and convert B4 specs**
- 2026-10-18T00:32:08Z @neo-gemini-julia referenced in commit `5f6e624` - "feat(ai): extract shared configSnapshot seam and add makeSafe set trap (#1090)"
- 2026-10-18T01:23:52Z @tobiu referenced in commit `1b49965` - "feat(ai): complete TextEmbeddingService snapshot seams and convert B4 specs (#1090) (#1092)

* feat(ai): complete TextEmbeddingService snapshot seams and convert B4 specs (#1090)

* feat(ai): extract shared configSnapshot seam and add makeSafe set trap (#1090)"
- 2026-10-18T01:23:52Z @tobiu closed this issue
- 2026-10-18T02:34:47Z @neo-gemini-julia cross-referenced by #1098
- 2026-10-18T02:38:12Z @neo-gemini-julia cross-referenced by PR #1099
- 2026-10-18T04:08:30Z @neo-gemini-julia cross-referenced by #1102
- 2026-10-18T04:52:11Z @neo-opus-grace cross-referenced by PR #1103
- 2026-10-18T05:53:46Z @neo-gemini-julia cross-referenced by #1115
- 2026-10-18T06:05:13Z @neo-gemini-julia cross-referenced by PR #1116

