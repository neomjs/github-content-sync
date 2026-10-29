---
id: 1338
title: The NL relocation composition proof cannot run in the Brain repo
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-28T22:48:10Z'
updatedAt: '2026-10-28T23:17:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1338'
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
closedAt: '2026-10-28T23:17:38Z'
---
# The NL relocation composition proof cannot run in the Brain repo

## Context

`test/playwright/unit/ai/services/memory-core/helpers/nlRelocationComposition.spec.mjs` is the composition proof that what the container admits as `nl-action-telemetry` graph nodes is what `GapInferenceEngine.inferNlActionDigest` reads. Its JSDoc says why it exists: each half stayed green on its own while the pair never met.

Its `beforeAll` spawns a child whose script imports `${rootDir}/src/Neo.mjs` and `${rootDir}/src/core/_export.mjs`, the engine's layout before the split. That path has never been tracked in this repository (`git log -- src/Neo.mjs` is empty; the spec arrived with 11552b08, #13). The child exits with `ERR_MODULE_NOT_FOUND` before it admits anything, so none of the spec's 11 tests reaches an assertion.

- CI: Brain Unit run 40679982173 on dev `05e42855` lists 4 of its 9 retained-suite reds from this spec. The 10-18 census on #201 (6245894428) had the same 4.
- Local, dev `05e42855`, with CI's install (`npm ci --ignore-scripts`, `npm rebuild better-sqlite3`): `npx playwright test -c test/playwright/playwright.config.unit.mjs <spec> --retries=0` → 6 failed, 5 did not run, stderr `ERR_MODULE_NOT_FOUND … neo-agent-brain/src/Neo.mjs`.

## The Architectural Reality

- The child runs `node --input-type=module -e <script>` with `cwd: rootDir`, and `rel(file)` builds every import as `path.join(rootDir, file)`. The Brain-owned imports (`ai/…`) are correct; only the two engine imports point into a tree the Brain does not have.
- The engine is a dependency (`neo.mjs` in `package.json`). #1110 / PR #1111 fixed this class in `initTier1ConfigMigration.spec.mjs`: resolve `import.meta.resolve('neo.mjs/src/Neo.mjs')` and `'neo.mjs/src/core/_export.mjs'` in the parent and write those URLs into the generated source.
- The stack passes through `configTemplateResolver.mjs:100` (`nextResolve`). That hook only redirects `config.mjs` overlays, so it is not the cause.

## The Fix

Resolve both engine modules in the parent with `import.meta.resolve` and write the URLs into the child's script, as #1111 does. Leave the child's environment, its disposable graph and every assertion as they are.

The 11 assertions then execute here for the first time. A red that names real drift between admission and digest is what the spec exists to find: fix it in the owning layer if it fits this PR, otherwise file it and leave that arm red. No assertion changes to make it pass.

## Acceptance Criteria

- [ ] The spec imports the installed engine; no `${rootDir}/src/…` import remains.
- [ ] All 11 tests reach their assertions under the unit config with `--retries=0` (none "did not run").
- [ ] Each test is green, or its red has its own ticket; no assertion is weakened.
- [ ] On the PR, Brain Unit's base/head comparison drops this spec's reds and introduces none.

## Out of Scope

The other five retained-suite reds on `05e42855`: `FleetServerComposition`, the `GuideToolParity` neural-link arm, the `lintGuides` guide-series arm, `generateRosterOnboarding` and `DatabaseService.graphBackup`. They stay on #201.

## Related

Related: #201 (the retained-suite burn-down; this is its largest remaining cluster) · #1110 / #1111 (same class).

Live latest-open + closed sweep: latest 20 open Brain issues + closed search ("composition spec Neo.mjs Brain tree engine path") at 2026-10-28T22:47Z; closed #526 is the same split class on another surface (ontology edges), no equivalent. A2A: last 30 messages, no claim on this scope. Memory Core: recall miss on the problem nouns. Own assignments: #201 is the umbrella; no slice is open for this spec.
Structure map: N/A (no file added or moved).
Decision Record impact: none.

Ownership: self-assigned, as a slice of #201.

Origin Session ID: 192574ae-badb-4712-8cd2-9e80f5566d22
Retrieval Hint: "nlRelocationComposition child ERR_MODULE_NOT_FOUND src/Neo.mjs Brain tree"

## Timeline

- 2026-10-28T22:48:11Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-28T22:48:13Z @neo-opus-ada added the `bug` label
- 2026-10-28T22:48:13Z @neo-opus-ada added the `ai` label
- 2026-10-28T22:48:13Z @neo-opus-ada added the `testing` label
- 2026-10-28T22:50:55Z @neo-opus-ada cross-referenced by PR #1339
- 2026-10-28T22:53:46Z @neo-opus-ada cross-referenced by #1340
- 2026-10-28T22:57:13Z @neo-opus-ada cross-referenced by #201
- 2026-10-28T23:17:38Z @tobiu referenced in commit `6c29d88` - "fix(test): the NL relocation composition child imports the installed engine (#1338) (#1339)

The child's script imported ${rootDir}/src/Neo.mjs and src/core/_export.mjs, a layout this repository never had, so beforeAll died on ERR_MODULE_NOT_FOUND and none of the 11 tests reached an assertion. It now imports the installed engine through import.meta.resolve, as #1111 did for the Tier-1 migration fixtures; all 11 run and pass. One durable comment drops a review-action reference the archaeology gate flags."
- 2026-10-28T23:17:38Z @tobiu closed this issue

