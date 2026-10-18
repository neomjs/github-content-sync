---
id: 1090
title: Complete TextEmbeddingService snapshot seams and remaining B4 test slices
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - testing
  - architecture
assignees:
  - neo-gemini-pro
createdAt: '2026-10-17T22:38:29Z'
updatedAt: '2026-10-17T23:28:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1090'
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
# Complete TextEmbeddingService snapshot seams and remaining B4 test slices

## Context

unowned-rationale: Successor to #541/#1086 for the remaining B4 shared-config write concentrations and TextEmbeddingService seam retirement; parked with measured file rankings for the next maintainer pickup.

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-17T22:37:00Z`; no duplicate found (`#541` is closing via PR `#1086`).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-17T22:37:10Z`; no competing `[lane-claim]` or `[lane-intent]` on B4 shared-config write isolation.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"shared-config write findSharedConfigMutations TextEmbeddingService"`; surfaced `#541` and PR `#1086` Round-1 review (`5560361878`).
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me`; only `#541` and `#1008` assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/services/memory-core/TextEmbeddingService.mjs`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## The Problem

`#541` (resolved by PR `#1086`) established the `configSnapshot` / `createConfigSnapshot()` injection pattern on `TextEmbeddingService` and converted the largest single concentration of ADR-0019 B4 shared-config writes (`test/playwright/unit/ai/services/memory-core/TextEmbeddingService.retry.spec.mjs`, `80 -> 0` writes).

After `#1086`:
1. `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` reports `577 shared-config write(s) across 142 leaf/leaves` across `55` test files (`518` assignment, `59` `Object.assign`), led by:
   - `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.spec.mjs` (`65` writes)
   - `test/playwright/unit/ai/daemons/orchestrator/services/DeploymentStateBridgeService.spec.mjs` (`57` writes)
   - `test/playwright/unit/ai/daemons/orchestrator/Orchestrator.invariants.spec.mjs` (`45` writes)
   - `test/playwright/unit/ai/services/graph/GoldenPathSynthesizer.spec.mjs` (`44` writes)
   - `test/playwright/unit/src/evolution/RemDigestion.spec.mjs` (`44` writes)
   - `test/playwright/unit/ai/services/knowledge-base/VectorService.WorkVolumeBranching.spec.mjs` (`22` writes)
   - `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs` (`21` writes)
   - `test/playwright/unit/ai/services/github-workflow/IssueSyncer.spec.mjs` (`17` writes)
2. During Round-1 review of `#1086` (`5560361878`), `@neo-opus-ada` identified three concrete hardening items on `TextEmbeddingService`'s snapshot seam itself:
   - `test/playwright/unit/ai/services/memory-core/VectorService.batchFailureIsolation.spec.mjs` (`:164, :167, :223`) is the sole external consumer of `TextEmbeddingService.openAiCompatibleHostFn_`; moving it onto `configSnapshot` allows deleting `openAiCompatibleHostFn_` entirely.
   - `set configSnapshot(value)` does not currently reset `this.ollamaProvider = null`, so a native-Ollama test that sets `configSnapshot` after `#getOllamaProvider()` has already cached a `Neo.ai.provider.Ollama` instance would keep the prior host/model.
   - `createConfigSnapshot()` currently copies only enumerated leaves rather than cloning the whole consumed subtrees (`localModels.embedding`, `ollama`, `openAiCompatible`, `orchestrator.lms`, `orchestrator.providerReadiness`), which could let a future `TextEmbeddingService` read silently fall back to destructuring defaults (`unloadRetryCount = 3`, etc.) under snapshot mode.

## The Architectural Reality

- `ai/services/memory-core/TextEmbeddingService.mjs` owns `configSnapshot`, `beforeSetConfigSnapshot(value)`, and `createConfigSnapshot()`.
- `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs` still performs `21` shared-config writes against `aiConfig`, and `test/playwright/unit/ai/services/memory-core/VectorService.batchFailureIsolation.spec.mjs` still mutates `TextEmbeddingService.openAiCompatibleHostFn`.
- `ai/scripts/lint/check-aiconfig-test-mutation.mjs` exports `findSharedConfigMutations(content)`, which remains report-only until the remaining `55` test files are converted to isolated snapshot injection.

## The Fix

1. In `ai/services/memory-core/TextEmbeddingService.mjs`:
   - Clone the whole resolved subtrees (`localModels.embedding`, `ollama`, `openAiCompatible`, `orchestrator.lms`, `orchestrator.providerReadiness`) in `createConfigSnapshot()` via plain-object spread of resolved descriptors/leaves so newly added reads in those subtrees are automatically present on snapshots.
   - Reset `this.ollamaProvider = null` when `set configSnapshot(value)` changes the active snapshot so `#getOllamaProvider()` cannot serve a stale cached provider across snapshot transitions.
   - Delete `openAiCompatibleHostFn_` and read `this.#getConfig().openAiCompatible.host` directly at the `TextEmbeddingService` call sites after migrating `VectorService.batchFailureIsolation.spec.mjs` and `TextEmbeddingService.spec.mjs` onto `configSnapshot`.
2. Convert `test/playwright/unit/ai/services/memory-core/TextEmbeddingService.spec.mjs` (`21` writes) and `test/playwright/unit/ai/services/memory-core/VectorService.batchFailureIsolation.spec.mjs` onto `TextEmbeddingService.createConfigSnapshot()`, asserting `findSharedConfigMutations(source) === []`.
3. Convert the remaining B4 shared-config write concentrations across `test/**` (starting with `Orchestrator.spec.mjs`, `DeploymentStateBridgeService.spec.mjs`, `Orchestrator.invariants.spec.mjs`, `GoldenPathSynthesizer.spec.mjs`, and `RemDigestion.spec.mjs`) by `configSnapshot` injection, with no per-leaf `*Fn_` seams.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `TextEmbeddingService.configSnapshot` / `createConfigSnapshot()` (`ai/services/memory-core/TextEmbeddingService.mjs`) | ADR-0019 §4 B4 + `#541` | Clones whole consumed subtrees, resets cached `ollamaProvider` on snapshot change, and retires `openAiCompatibleHostFn_` | Throw `TypeError` via `beforeSetConfigSnapshot` on non-isolated inputs | JSDoc on `TextEmbeddingService.mjs` | Unit tests in `TextEmbeddingService.spec.mjs`, `TextEmbeddingService.retry.spec.mjs`, `VectorService.batchFailureIsolation.spec.mjs` |
| `findSharedConfigMutations` (`ai/scripts/lint/check-aiconfig-test-mutation.mjs`) | ADR-0019 §4 B4 | Per-spec assertions pin `0` shared-config writes as each concentration converts | Report-only global count until last file converts | JSDoc in `check-aiconfig-test-mutation.mjs` | `npm run test-unit` + `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] `createConfigSnapshot()` on `TextEmbeddingService` copies full resolved plain-object subtrees for `localModels.embedding`, `ollama`, `openAiCompatible`, `orchestrator.lms`, and `orchestrator.providerReadiness`.
- [ ] `set configSnapshot(value)` resets `this.ollamaProvider = null` when the snapshot changes (verified by a unit test on the native Ollama path).
- [ ] `VectorService.batchFailureIsolation.spec.mjs` and `TextEmbeddingService.spec.mjs` use `configSnapshot` (`0` shared-config writes in `findSharedConfigMutations`), and `openAiCompatibleHostFn_` is removed from `TextEmbeddingService.mjs`.
- [ ] The remaining B4 shared-config write concentrations across `test/**` are converted or decomposed into tracked per-service slices until `findSharedConfigMutations` can be promoted from report-only to gating.

## Out of Scope

- Changing `Neo.mjs`'s reactive config setter (`setRaw` before `beforeSet*`), which is tracked separately in `neomjs/neo`.

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`; tests must inject an isolated `configSnapshot` rather than proliferating one function config per leaf.

## Related

- Succeeds: `#541` / PR `#1086`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "TextEmbeddingService configSnapshot openAiCompatibleHostFn findSharedConfigMutations B4"

## Timeline

- 2026-10-17T22:38:30Z @neo-gemini-pro added the `enhancement` label
- 2026-10-17T22:38:30Z @neo-gemini-pro added the `ai` label
- 2026-10-17T22:38:31Z @neo-gemini-pro added the `refactoring` label
- 2026-10-17T22:38:31Z @neo-gemini-pro added the `testing` label
- 2026-10-17T22:38:31Z @neo-gemini-pro added the `architecture` label
- 2026-10-17T22:38:43Z @neo-gemini-pro cross-referenced by #541
- 2026-10-17T22:40:03Z @neo-gemini-pro cross-referenced by PR #1086
- 2026-10-17T23:28:54Z @neo-gemini-pro assigned to @neo-gemini-pro

