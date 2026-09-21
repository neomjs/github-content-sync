---
id: 257
title: Receive ADR-0019 guards into Brain before the Engine re-pin
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - regression
  - architecture
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-30T19:28:06Z'
updatedAt: '2026-08-30T20:40:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/257'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 194
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 184 Align Brain''s Engine pin with post-split consumers'
closedAt: '2026-08-30T20:40:31Z'
---
# Receive ADR-0019 guards into Brain before the Engine re-pin

## Context

Brain PR #255 merged at `b1bc6101f26a8914db53587cd858ac66191f640d`, removing the last Engine-root projections. The next cutover dependency, #184, advances Brain from pre-split Engine `21da6802…` to Institution's post-split pin `17b59aad…`.

A fresh exact candidate install at that pin succeeds and contains zero `ai/**`, but the full Brain unit collection aborts before tests run:

```text
Cannot find module 'node_modules/neo.mjs/buildScripts/util/check-aiconfig-test-mutation.mjs'
imported from test/playwright/unit/ai/scripts/diagnostics/printAiConfig.spec.mjs
```

The same pin also removes `check-aiconfig-antipatterns.mjs`. Current Brain source still reads both from the installed Engine in `ai/scripts/lint/lint-config-template-ssot.mjs`.

## The Problem

Engine PR neomjs/neo#17806 correctly deleted received Brain implementation, including both ADR-0019 guards and their specs. Brain received neither guard before deleting its copied `buildScripts` tree in #198. The stale pre-split Engine dependency masked that omission.

This is not one missing test helper:

- `check-aiconfig-test-mutation` is ADR 0019's B4 safety-critical enforcement anchor and owns the parser-grade `codeMask`;
- `check-aiconfig-antipatterns` consumes that mask and owns A1/A5/B3 enforcement;
- `lint-config-template-ssot` reads both sources to verify the ADR's two-way enforcement catalog;
- `printAiConfig.spec.mjs` imports `findDbPathMutations` from the mutation guard;
- the old Engine workflows executed both guards, while Brain's current workflow only runs `lint-config-template-ssot`.

Advancing the Engine pin without receiving them would either break CI or tempt deletion of safety-critical enforcement.

## The Architectural Reality

ADR 0019 makes these guards Brain-owned Agent OS configuration enforcement. ADR 0040 requires Brain → Engine dependency direction for Engine surfaces; the Engine package must never remain a fallback distribution channel for Brain implementation.

Canonical homes are already established:

- one-shot Brain lints: `ai/scripts/lint/`;
- their unit contracts: `test/playwright/unit/ai/scripts/lint/`;
- the consolidated trigger surface: `.github/workflows/config-template-ssot-lint.yml`, which already watches `ai/**/*.mjs` and `test/**/*.mjs`.

Pre-Flight (structural fast-path): `ai/scripts/lint/check-aiconfig-test-mutation.mjs` and `check-aiconfig-antipatterns.mjs` match the existing check/lint siblings in `ai/scripts/lint/`; their relocated specs match `lintConfigTemplateSsot.spec.mjs`. No novel directory or map update is introduced. `npm run --silent ai:structure-map -- --files --loc` passed before ticket creation.

## The Fix

1. Receive the two guards from the last pre-removal Engine source (`c623b2f63c^`) into `ai/scripts/lint/`, preserving provenance and adapting only repository-root/path assumptions.
2. Relocate their mutation-sensitive specs into `test/playwright/unit/ai/scripts/lint/`.
3. Point `lint-config-template-ssot.mjs` and `printAiConfig.spec.mjs` at the Brain-local authority.
4. Execute both guards from the existing Config Template SSOT workflow; do not recreate two workflow wrappers with overlapping triggers.
5. Prove a fresh post-split Engine install collects and runs the guards without any `node_modules/neo.mjs/buildScripts/util/check-aiconfig*` dependency.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| B4 mutation guard | ADR 0019 §3/§4 | Brain-local guard scans `test/**` and exports the parser-grade mask/mutation detector | Parse failure or mutation fails closed; no Engine fallback | source JSDoc | restored mutation suite + induced mutation |
| A1/A5/B3 guard | ADR 0019 catalog | Brain-local guard consumes the local mask and scans Brain `ai/**` | Unknown/unclassified violations fail | source JSDoc | restored guard suite + live-tree run |
| ADR enforcement registry | `lint-config-template-ssot.mjs` | Reads both Brain-local guard sources for two-way catalog ownership | Missing guard source makes the lint red | none | focused registry tests |
| Config lint CI | `config-template-ssot-lint.yml` | Runs all three ADR-0019 guards under one already-complete watch surface | Any guard failure is a failed required job | workflow comments | workflow contract + hosted CI |
| Diagnostic CLI test | `printAiConfig.spec.mjs` | Imports the local `findDbPathMutations` authority | Missing local guard fails collection | none | focused unit test |

## Decision Record impact

`aligned-with ADR 0019 and ADR 0040` — restores the enforcement ADR 0019 names while removing the forbidden Brain-implementation-via-Engine-package dependency.

## Acceptance Criteria

- [ ] Both ADR-0019 guard modules live under `ai/scripts/lint/` with provenance-preserving JSDoc and no Engine-root implementation import.
- [ ] Their mutation-sensitive unit contracts live under `test/playwright/unit/ai/scripts/lint/` and cover parser-mask, mutation, allowlist, CLI-red, and live-tree behavior.
- [ ] `lint-config-template-ssot.mjs` and `printAiConfig.spec.mjs` consume the Brain-local guard.
- [ ] Config Template SSOT CI invokes the mutation guard, antipattern guard, and catalog lint under its existing complete path filters.
- [ ] An induced B4 mutation and an induced A1/A5/B3 antipattern each fail the owning guard; clean controls pass.
- [ ] Fresh `npm ci` against post-split Engine `17b59aad…` collects/runs the focused suites with zero `node_modules/neo.mjs/buildScripts/util/check-aiconfig*` dependency.
- [ ] #184 remains a dependency-coordinate-only PR after this prerequisite lands.

## Out of Scope

- Changing AiConfig leaves, Provider resolution, or ADR 0019 policy.
- Advancing the Engine pin (#184).
- Restoring unrelated Engine build guards or copied `buildScripts`.
- Repairing the broader retained-red suite (#201).
- Reintroducing the two deleted Engine workflow wrappers.

## Avoided Traps

- **Delete the consumers:** loses the safety-critical guard while making collection green.
- **Keep the pre-split Engine pin:** preserves a hidden implementation dependency and blocks the container milestone.
- **Copy the entire old buildScripts tree:** recreates #198's compatibility projection.
- **Two new workflows:** duplicates one complete trigger/watch surface into three partially overlapping authorities.

## Related

Parent: #194  
Blocks: #184  
Related: #198 · PR #255 · neomjs/neo#17791 · neomjs/neo PR #17806 · #201

Live latest-open sweep: checked the latest 20 open Brain issues created-descending at 2026-08-30T19:26:00Z; no equivalent found.  
A2A in-flight sweep: checked the latest 30 messages across all read states at the same time; no overlapping guard-receive claim found.

Origin Session ID: `4d087bc8-dfee-4b33-9826-b99a7edee0b5`

Retrieval Hint: `Brain ADR-0019 guard receive check-aiconfig-test-mutation Engine pin 17b59aad`
Retrieval Hint: Engine commit `c623b2f63c^`

## Timeline

- 2026-08-30T19:28:06Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `bug` label
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `regression` label
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `architecture` label
- 2026-08-30T19:28:08Z @neo-gpt-emmy added the `build` label
- 2026-08-30T19:28:09Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-30T19:48:15Z @neo-gpt-emmy cross-referenced by PR #259
- 2026-08-30T19:50:43Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-30T20:40:31Z @tobiu referenced in commit `894f1a0` - "Merge pull request #259 from neomjs/codex/257-receive-aiconfig-guards

fix(lint): receive ADR-0019 guards into Brain (#257)"
- 2026-08-30T20:40:32Z @tobiu closed this issue
- 2026-08-30T20:55:04Z @neo-opus-ada cross-referenced by #263

