---
id: 1346
title: GuideToolParity reads NeuralLink.md from a current installed engine
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T13:22:26Z'
updatedAt: '2026-10-29T13:41:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1346'
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
blockedBy:
  - '[ ] 1356 The KB core corpus stops routing ConceptSource over the engine''s deleted concepts'
  - '[ ] 474 The KB projects admitted concepts from the plane store'
blocking: []
---
# GuideToolParity reads NeuralLink.md from a current installed engine

## Context

#201's retained suite has three reds left (comment 6482850355). One is `test/playwright/unit/ai/mcp/validation/GuideToolParity.spec.mjs`. neomjs/neo#19855 did the Engine half (PR neomjs/neo#20168, merged 2026-10-29 as `cc24c59e0d`): neo's capability-matrix copy is gone, and `learn/agentos/NeuralLink.md` states no operation count. This ticket is the Brain half that #19855 records.

## The Problem

The arm fails before its first assertion, and would still fail after re-pointing:

1. It reads `learn/agentos/NeuralLink.md` from the Brain root (`path.join(repoRoot, guide)`, spec line 63). The guide is Engine-owned and the Brain has no copy: `ENOENT` in Brain Unit run 40792765558 (`dev@b98d30ea`).
2. Its count assertion is unconditional, `` toContain(`${exposed.size} operation IDs`) `` (line 83), although the header comment (lines 11–14) asks only that an explicit count claim match.
3. The installed engine is the `7f16355a` tarball (2026-09-25). Its `NeuralLink.md:205` still says "59 operation IDs", while the Brain's `openapi.yaml` has 60 (`drive_drag`, #204). Re-pointing without a pin move changes the error but leaves the arm red.

The pin is also 480 engine commits behind `cc24c59e0d`. 44 of them are fixes touching engine files the Brain loads (`npm run ai:pin-drift -- --ref cc24c59e0d`), including neomjs/neo#19269 (Provider enumeration), neomjs/neo#19852 (a throwing `beforeSet`) and neomjs/neo#20141 (late config writes on a destroyed instance). The Institution's lock installs engine `95e63df6` (2026-10-28) for the organism it ships. So the Brain's suite certifies a different engine than the one that runs.

**Measured 2026-10-29: the pin move turns five more retained tests red.** Each one reads the 09-25 engine. Classification is the same instrument on base: the full suite at the new pin, then the same tests on base in the same clone, then a head-only rerun in isolation. Eight tests failed on head only; three of those pass in isolation (load flakes), and five fail deterministically:

| test | reads | disposition |
|---|---|---|
| `discussionRoutingDisposition` › two corpus arms | four discussions under the engine's `resources/content`, which neomjs/neo#19322 deleted on 09-29 | verbatim excerpts from `neomjs/github-content-sync`, here |
| `ConceptIngestor` › the complete repository fixture | the stale `DreamService.mjs` ontology row that neomjs/neo#19236 fixed | expect no findings, here |
| `CoalescingEngineService` › the inherited-leaves regression | the Provider enumeration trap that neomjs/neo#19269 fixed | keep the by-reference check, drop the trap pin, here |
| KB `DatabaseService.sync` › the exact Engine pin… | **production** `createKnowledgeBase`: `coreCorpusProfilePlan.mjs:97` routes `ConceptSource` over the engine's deleted `resources/content/concepts` | **blocker**: #474 AC-4 retires that route |

## The Architectural Reality

- The registry (`ai/mcp/server/neural-link/openapi.yaml`) belongs to the Brain and the guide to the Engine, per the #1117 decision under `adr:agentos-extraction-topology`.
- Sibling precedent for reading Engine-owned files: `provisioningTemplates.spec.mjs`'s `readEngineFile`, which uses `new URL(import.meta.resolve('neo.mjs/<path>'))`. `neo.mjs` has no `exports`, so the bare specifier reaches `learn/`.
- The pin is a commit tarball (`https://github.com/neomjs/neo/archive/<sha>.tar.gz`), set in #482 so the engine's lifecycle stays out of dependency builds. Between the two pins, neo's lifecycle scripts only gain the pack-time `prepack`/`postpack`. `prepare` is unchanged, and npm does not run it for tarball dependencies.
- Brain Unit runs every retained spec on head and base (#650), so its `introduced` count measures the pin move.
- The KB compiles the engine corpus at the pin revision. The sync spec asserts the manifest's engine revision equals the pin. So the route blocker is a production dependency, not only a test one. `embedKnowledgeBase` is additive ("routine stale deletion remains disabled"), so retiring the route keeps the legacy `type: concept` vectors that #419 AC-4 retains.

## The Fix

1. `package.json` + `package-lock.json`: the `neo.mjs` tarball moves from `7f16355a` to `cc24c59e0d`, neo `dev` at the #20168 merge.
2. `GuideToolParity.spec.mjs`: the guide is read from the installed engine (the `readEngineFile` shape) and the OpenAPI from the Brain root. The count check runs only when the guide states `<N> operation IDs`, and then N must equal the operationId count. The count extraction gets its own test, so the conditional cannot pass vacuously.
3. The three test adaptations in the table above.
4. Lands after the `ConceptSource` route retirement (#474 AC-4, or a leaf split from it).

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Evidence |
|---|---|---|---|---|
| guide read in `GuideToolParity` | the Engine owns `learn/agentos/NeuralLink.md` | read via `import.meta.resolve('neo.mjs/learn/agentos/NeuralLink.md')` | a missing engine file stays a red `ENOENT`, never a skip | arm green at the new pin |
| operation-count claim | the operationIds in the Brain's `openapi.yaml` (60 at `dev`) | asserted only when the guide states a count, and then exactly | a stated stale count fails | extraction test: a stated 59 against 60 fails, no claim passes |
| `neo.mjs` pin | `package-lock.json` `resolved` | the `cc24c59e0d` tarball, with npm's integrity | installed must equal the lock | `ai:pin-drift` prints `installed … ✓` |
| corpus samples for the routing classifier | `neomjs/github-content-sync`, the corpus owner since neomjs/neo#19322 | verbatim excerpts in a fixture beside the spec, with source path and revision | an excerpt that classifies differently from its whole body is not an excerpt | probe: each excerpt's result equals the whole body's |

Decision Record impact: aligned-with `adr:agentos-extraction-topology`.

## Acceptance Criteria

- [ ] AC-1: `GuideToolParity` reads the guide from the installed engine and passes at the new pin.
- [ ] AC-2: a stated count that differs from the operationId count fails and no stated count passes, proven by a test of the extraction as well as the live guide.
- [ ] AC-3: `package.json` and the lock name `cc24c59e0d`, and `npm run ai:pin-drift` reports the installed engine equal to the lock.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds, with `GuideToolParity` among the fixed.
- [ ] AC-5 (post-merge): #201's dated count drops to two reds.

## Out of Scope

- `graphBackup` and `lintGuides`, #201's other two reds, which need separate decisions.
- The `ConceptSource` route itself (#474), and Brain code changes that adopt the 44 engine fixes beyond what keeps the suite green.
- Following `#dev` the way the Institution does. The tarball pin is #482's choice.

## Avoided Traps

- Re-pointing without the pin move: the old pin's guide states 59, so the arm stays red for a different reason.
- Copying `NeuralLink.md` into the Brain, or deleting the arm. The guard exists to keep the Engine's guide honest against the Brain's registry.
- Skipping an arm when its engine file is absent, or tolerating a missing `ConceptSource` directory as zero concepts.
- Fixturing the four discussions whole (~93KB of prose) when four verbatim excerpts (~3KB) classify identically.

## Related

Parent: #201. Engine half: neomjs/neo#19855, neomjs/neo#20168. Receive decision: #1117. Pin form: #482. Blocker: #474. Sibling slices: #1110, #1112, #1125.

Live latest-open + closed sweep: latest 20 open Brain issues + a closed search (GuideToolParity, NeuralLink.md, engine pin) at 2026-10-29T13:20:30Z, re-checked at 13:22:00Z; the hits (#1117, #1125, #184, #482-era) are the decision and sibling slices, with no equivalent. A2A claim sweep: the latest 30 messages, all states, with no claim. Own assignments: #201 (parent). Structure map: N/A, since this adds or moves no `.mjs` file.

Retrieval Hint: "GuideToolParity ENOENT NeuralLink.md installed engine; Brain neo.mjs pin 7f16355a to cc24c59e0d; five pin-forced reds; ConceptSource route blocks the pin move"


## Timeline

- 2026-10-29T13:22:26Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T13:22:27Z @neo-opus-ada added the `bug` label
- 2026-10-29T13:22:27Z @neo-opus-ada added the `ai` label
- 2026-10-29T13:22:27Z @neo-opus-ada added the `testing` label
- 2026-10-29T13:41:46Z @neo-opus-ada marked this issue as being blocked by #474
- 2026-10-29T13:42:23Z @neo-opus-ada cross-referenced by #1347
- 2026-10-29T13:57:10Z @neo-opus-ada cross-referenced by #1353
- 2026-10-29T14:10:24Z @neo-opus-ada cross-referenced by #1356
- 2026-10-29T14:10:33Z @neo-opus-ada marked this issue as being blocked by #1356

