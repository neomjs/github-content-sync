---
id: 1115
title: Convert remaining 151 B4 shared-config writes across 31 service and MCP test files
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - testing
assignees: []
createdAt: '2026-10-18T05:53:45Z'
updatedAt: '2026-10-18T18:17:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1115'
author: neo-gemini-julia
commentsCount: 1
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
closedAt: '2026-10-18T18:17:16Z'
---
# Convert remaining 151 B4 shared-config writes across 31 service and MCP test files

unowned-rationale: Successor to #1102 (#1098 / #1091 / #1090 / #541) for the final 151 B4 shared-config writes across 31 test files after the Orchestrator daemon conversion; parked with measured file and leaf rankings for the next maintainer pickup.

- **Live latest-open sweep:** Checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-18T05:53:12Z` (`#1112`, `#1110`, `#1102`, `#1101`, `#1081`, `#1048`, `#1047`, `#1014`, `#1013`, `#1012`, `#1010`, `#1005`, `#979`, `#978`, `#977`, `#969`, `#957`, `#936`, `#933`, `#904`); no duplicate found (`#1102` is scoped to the `ai/daemons/orchestrator/` `191`-write conversion across `6` test files).
- **A2A in-flight claim sweep:** Checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-18T05:53:00Z`; no competing claim on converting the remaining `31` MCP server, Knowledge Base, Memory Core, and service test files.
- **Memory Core rationale sweep:** Queried `query_raw_memories` for `"check-aiconfig-test-mutation B4 shared-config write concentrations SessionService DestructiveOperationGuard KnowledgeBase"` at `2026-10-18T05:53:01Z`; surfaced `#541`, `#1086`, `#1090`, `#1092`, `#1091`, `#1096`, `#1097`, `#1099`, `#1098`, `#1103`, and `#1102`.
- **Own-assignment sweep:** Checked `gh issue list --repo neomjs/neo-agent-brain --state open --assignee @me` at `2026-10-18T05:53:02Z`; only `#1102` assigned.
- **Structure Map gate:** Verified via `npm run ai:structure-map -- --files --loc` (`ai/services/memory-core/SessionService.mjs`, `ai/mcp/server/shared/services/DestructiveOperationGuard.mjs`, `ai/services/knowledge-base/`, `ai/scripts/lint/check-aiconfig-test-mutation.mjs`).

## Context

`#541` (PR `#1086`), `#1090` (PR `#1092`), `#1091` (PR `#1099`), `#1098` (PR `#1103`), and `#1102` converted `TextEmbeddingService` (`2` specs), `VectorService` (`7` specs), `GoldenPathSynthesizer` (`1` spec), `RemDigestion` (`2` specs), `github-workflow` services (`6` specs), and `ai/daemons/orchestrator/` (`6` specs) to isolated `configSnapshot` / `createConfigSnapshot()` injection backed by `ai/services/shared/configSnapshot.mjs`, while `#1096` (PR `#1097`) introduced the per-file ceiling ratchet (`ai/scripts/lint/shared-config-write-ceilings.json`), reducing global B4 shared-config writes from `655` to `151` across `31` remaining test files (`0` `Object.assign` writes remaining repo-wide).

## The Problem

`node ai/scripts/lint/check-aiconfig-test-mutation.mjs` still reports `151 shared-config write(s) across 45 leaf/leaves (151 assignment, 0 Object.assign)` across `31` remaining test files in `ai/scripts/lint/shared-config-write-ceilings.json`:
- Top remaining test file concentrations:
  - `test/playwright/unit/ai/services/memory-core/SessionSummarization.spec.mjs` (`13` writes)
  - `test/playwright/unit/ai/mcp/server/shared/services/DestructiveOperationGuard.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/knowledge-base/source/ApiSource.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/rem-observability.spec.mjs` (`12` writes)
  - `test/playwright/unit/ai/services/knowledge-base/source/SkillSource.spec.mjs` (`10` writes)
  - `test/playwright/unit/ai/mcp/server/github-workflow/ConfigCompleteness.spec.mjs` (`8` writes)
  - `test/playwright/unit/ai/mcp/server/knowledge-base/IngestSourceFilesTool.spec.mjs` (`6` writes)
  - `test/playwright/unit/ai/mcp/server/knowledge-base/logger.spec.mjs` (`6` writes)
  - `test/playwright/unit/ai/mcp/server/memory-core/logger.spec.mjs` (`6` writes)
  - `test/playwright/unit/ai/mcp/server/neural-link/logger.spec.mjs` (`6` writes)
  - `test/playwright/unit/ai/services/knowledge-base/KnowledgeBase.TenantIsolation.spec.mjs` (`6` writes)
  - `test/playwright/unit/ai/services/memory-core/MailboxService.spec.mjs` (`6` writes)
- Top remaining leaf targets:
  - `.data.debug` (`15`), `.neoRootDir` (`12`), `.hierarchyPath` (`11`), `.transport` (`7`), `.data.handoffFilePathTest` (`7`), `.data.logPath` (`6`), `.data.logLevel` (`5`), `.engines.chroma.useUnitTestDatabase` (`4`), `.engines.chroma.useTestHarness` (`4`), `.projectRoot` (`4`).

## The Architectural Reality

- `ai/services/shared/configSnapshot.mjs` (`cloneConfigSubtree`, `validateConfigSnapshot`), `ai/services/shared/serviceProxy.mjs` (`makeSafe` `set` trap forwarding), and `ai/scripts/lint/shared-config-write-ceilings.json` provide the canonical ADR-0019 §4 B4 test-isolation seam and per-file ratchet.
- `SessionService`, `DestructiveOperationGuard`, Knowledge Base source services (`ApiSource`, `SkillSource`, `RawRepoSource`, `AdrSource`, `ConceptSource`), `MailboxService`, and MCP server loggers still read module-imported `aiConfig` / server config directly in unit tests that mutate shared config leaves.

## The Fix

1. Extend the `configSnapshot` / `beforeSetConfigSnapshot` / `createConfigSnapshot()` pattern (using `ai/services/shared/configSnapshot.mjs`) to the remaining target services (`SessionService`, `DestructiveOperationGuard`, KB sources, `MailboxService`, MCP loggers, etc.).
2. Convert the remaining `31` test files (`151` shared-config writes) to inject isolated snapshots and run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` to drop their entries from `ai/scripts/lint/shared-config-write-ceilings.json`.
3. Once all test files reach `0` shared-config writes, retire `ai/scripts/lint/shared-config-write-ceilings.json` so any shared-config write in `test/**` fails the lint gate unconditionally.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `findSharedConfigMutations` (`ai/scripts/lint/check-aiconfig-test-mutation.mjs`) | ADR-0019 §4 B4 + `#508` + `#1096` | Per-file shared-config write counts ratchet down to `0` across the remaining `31` files (`151` writes) | Per-file ceiling ratchet (`#1096`) blocks regressions on clean/ratcheted files | JSDoc in `check-aiconfig-test-mutation.mjs` | `npm run test-unit` + `node ai/scripts/lint/check-aiconfig-test-mutation.mjs` |

## Decision Record impact

`aligned-with ADR 0019`

## Acceptance Criteria

- [ ] Convert the remaining `31` test files (`151` shared-config writes) across `SessionService`, `DestructiveOperationGuard`, KB sources, `MailboxService`, MCP loggers, and remaining specs to isolated `configSnapshot` injection with no per-leaf `*Fn_` seams.
- [ ] Run `node ai/scripts/lint/check-aiconfig-test-mutation.mjs --tighten-ceilings` and retire the non-zero ceiling table once `0` shared-config writes remain across `test/**`.

## Out of Scope

- `TextEmbeddingService`, `VectorService`, `GoldenPathSynthesizer`, `RemDigestion`, `github-workflow`, and `ai/daemons/orchestrator/` specs completed in `#541` (`#1086`), `#1090` (`#1092`), `#1091` (`#1099`), `#1098` (`#1103`), and `#1102`.

## Avoided Traps

- **Adding per-leaf `*Fn_` seams:** Rejected per `#541`, `#1090`, `#1091`, `#1098`, and `#1102`; tests must inject an isolated `configSnapshot` rather than proliferating per-leaf function hooks.

## Related

- Succeeds: `#541` / PR `#1086`, `#1090` / PR `#1092`, `#1091` / PR `#1099`, `#1098` / PR `#1103`, `#1102`
- Related: `#508`, `#1096` / PR `#1097`

Origin Session ID: ae04cf6d-3b43-4898-8ee7-957f369c3ba5
Retrieval Hint: "findSharedConfigMutations B4 151 shared-config writes 31 test files SessionService DestructiveOperationGuard"


## Timeline

- 2026-10-18T05:53:47Z @neo-gemini-julia added the `enhancement` label
- 2026-10-18T05:53:47Z @neo-gemini-julia added the `ai` label
- 2026-10-18T05:53:47Z @neo-gemini-julia added the `refactoring` label
- 2026-10-18T05:53:47Z @neo-gemini-julia added the `testing` label
- 2026-10-18T06:05:13Z @neo-gemini-julia cross-referenced by PR #1116
- 2026-10-18T09:03:04Z @neo-gemini-julia cross-referenced by PR #1135
- 2026-10-18T09:21:33Z @neo-gemini-julia referenced in commit `7350c59` - "feat(ai): convert remaining 151 service and MCP B4 shared-config writes across 31 test files to configSnapshot (#1115)"
- 2026-10-18T17:16:20Z @neo-gemini-julia referenced in commit `43a3fd4` - "fix(ai): deep-clone queryCandidatePools.types in QueryService snapshot (#1115)"
- 2026-10-18T17:42:10Z @neo-gemini-julia referenced in commit `3bb413e` - "feat(ai): convert remaining 151 service and MCP B4 shared-config writes across 31 test files to configSnapshot (#1115)"
- 2026-10-18T17:42:11Z @neo-gemini-julia referenced in commit `b640576` - "fix(ai): deep-clone queryCandidatePools.types in QueryService snapshot (#1115)"
### @neo-gemini-julia - 2026-10-18T18:17:16Z

Closing as not planned per @tobiu's architectural review: `configSnapshot` violates ADR-0019 (`AiConfig` is a reactive hierarchical `Neo.state.Provider` SSOT). Follow-up tickets will revert/remove `configSnapshot` across `ai/` and `test/`.

- 2026-10-18T18:17:17Z @neo-gemini-julia closed this issue

