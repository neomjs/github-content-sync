---
id: 888
title: SyncService stops deriving Portal indexes and SEO
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-05T13:18:39Z'
updatedAt: '2026-10-05T13:58:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/888'
author: neo-opus-ada
commentsCount: 0
parentIssue: 459
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T13:58:56Z'
---
# SyncService stops deriving Portal indexes and SEO

## Context

This is a sub of #459, covering the SyncService row Grace added there ([5994442678](https://github.com/neomjs/neo-agent-brain/issues/459#issuecomment-5994442678)).

`neomjs/neo#19408`, merged 2026-10-05 13:11Z, makes the engine's `rebuildContentIndexesAndSeo` require `corpusRoot`. The Brain calls it with `{root: aiConfig.projectRoot}` alone (`ai/services/github-workflow/SyncService.mjs:74-77`), so the call throws once the Brain's `neo.mjs` pin moves past that merge. Brain CI cannot see this:
- every unit test that reaches the call replaces it: the `beforeEach` of `SyncService.Stage2.spec.mjs`, and `syncGithubWorkflow.spec.mjs`;
- `.github/dependabot.yml` records that the tarball pin is a manual move.

## The Problem

Passing `corpusRoot` would keep alive a step that no automated system runs.

- **The one scheduled caller does not derive.** `neomjs/github-content-sync`'s `publish-corpus.yml` runs the CLI with `--corpus-only`. That calls `emitConversationCorpus`, which passes `deriveContent: false`. The workflow refuses to run without that mode, because full mode "derives Portal artifacts and pushes issues back to GitHub".
- **Nothing else calls the CLI.** Scanned 2026-10-05: no workflow in neo, neo-agent-brain, neo-agent-institution, pages or devindex invokes it. The neomjs.com server copies `node_modules/neo.mjs/resources/content` and never runs it.
- **The CLI docblock is stale.** `syncGithubWorkflow.mjs:51` says the scheduled pipeline runs `--emit-only`; it runs `--corpus-only`.
- **The manual modes derive from a tree nobody has.** `runFullSync()` and `--emit-only` emit into `issueSync.contentRoot`. Without an override that is `<cwd>/resources/content` (`ai/mcp/server/github-workflow/configBase.mjs:8-9`, `:337`).
  - The Brain tracks no `resources/`.
  - An engine checkout lost its mirror in `neomjs/neo#19322`.
  - With `NEO_MCP_GITHUB_CONTENT_ROOT` set, the emit goes to the override, but the derive still reads the working directory.
- **The operator does not use the manual mode's Portal output.** Asked 2026-10-05, the answer was: "no. there should be no manual runs anyway inside automated systems."

Portal derivation now belongs to the Portal's own deploy: `neomjs/pages#14` has its step 7 pass `--corpus-root` and `--release-notes` to the engine's orchestrator.

## The Architectural Reality

Design authority: `SyncService#emitConversationCorpus`'s JSDoc, "Portal/SEO output, local-to-GitHub issue pushes and git publication stay outside this producer-only boundary." The producer that the scheduled pipeline runs already excludes derivation. This ticket removes the last Brain path that performs it.

- `SyncService.rebuildContentIndexesAndSeo()` (`:70-78`) imports `neo.mjs/buildScripts/docs/rebuildContentIndexesAndSeo.mjs`.
- `emitGeneratedContentAndDerive({pushLocalChanges, deriveContent})` (`:131`) calls it when `deriveContent` is true (`:325-326`). `emitConversationCorpus` passes `false` (`:371-372`). `runFullSync` (`:571`, `:575`) and the CLI's `--emit-only` (`syncGithubWorkflow.mjs:230`) take the default `true`.
- Tests:
  - `SyncService.Stage2.spec.mjs` stubs the derive in `beforeEach` and asserts its ordering and failure arms.
  - `syncGithubWorkflow.spec.mjs:130` stubs it.
  - `SyncService.RechunkOrdering.spec.mjs` (#298) anchors its ordering on the derive call.

## The Fix

- Delete `rebuildContentIndexesAndSeo` and the `deriveContent` option. Rename `emitGeneratedContentAndDerive` to `emitGeneratedContent`, and update its callers (`runFullSync`, `emitConversationCorpus`, the CLI) and its JSDoc.
- Correct `syncGithubWorkflow.mjs`'s docblock: the scheduled pipeline runs `--corpus-only`, and `--emit-only` and the default mode are manual. Neither derives Portal artifacts.
- Tests:
  - Drop the derive stubs and the derive arms.
  - Re-anchor `RechunkOrdering` on the property that survives: each corpus facet is re-chunked exactly once inside the emitter, before the emitter returns.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `SyncService.emitGeneratedContentAndDerive` (→ `emitGeneratedContent`) | this ticket; `emitConversationCorpus` JSDoc | renamed; no `deriveContent` option; never derives | none: every caller is updated in the same PR | the method's JSDoc | Stage2 and RechunkOrdering specs |
| `SyncService.rebuildContentIndexesAndSeo` | this ticket | removed | none | none | AC-1 predicate |
| `npm run ai:sync-github-workflow`, default mode and `--emit-only` | operator answer, 2026-10-05 | emit (and, in the default mode, push), with no Portal derivation | the Portal's deploy derives (`neomjs/pages#14`) | `syncGithubWorkflow.mjs` docblock | `syncGithubWorkflow.spec.mjs` |
| `SyncService` auto-push allowlist (`generatedSyncPaths`) | this ticket (a PR delta) | commits only the emitted corpus paths, with no `apps/portal/*` entry | none: nothing writes those paths once the derive is gone | none (a module constant) | `SyncService.Stage2.spec.mjs` allowlist arm |

## Acceptance Criteria

- [ ] **AC-1:** `git grep -nE "rebuildContentIndexesAndSeo|deriveContent" -- ai` returns nothing.
- [ ] **AC-2:** `--corpus-only` is unchanged. `--emit-only` and the default mode keep their emit and push behavior, without a derive. Covered by the existing `syncGithubWorkflow` and Stage2 arms, minus the derive arms.
- [ ] **AC-3:** `SyncService.RechunkOrdering.spec.mjs` still fails when a corpus facet's re-chunk is removed or duplicated. Red-checked in the PR.
- [ ] **AC-4:** `check-aiconfig-antipatterns` and `lint-config-template-ssot` pass, and no use-site default is added.

## Out of Scope

- **The manual modes themselves.** Without the derive, `--emit-only` still writes the single-origin layout from before `neomjs/neo#19322` under `<cwd>/resources/content`, and the default mode also pushes local issue edits to GitHub. Whether either should survive the operator's "no manual runs inside automated systems" is a separate decision. This ticket removes only the step that breaks on the pin move.
- The rest of #459: the Fleet `contentRoot`, the github-workflow `contentRoot` fallback, and the ingestion and KB typing rows.
- `learn/agentos/GitHubWorkflow.md`'s orchestration list. It does not name the derive, and its other staleness is not this ticket's.
- Moving the Brain's `neo.mjs` pin.

## Avoided Traps

- **Passing `corpusRoot` into the derive.** It would keep a step no automated system runs, couple the Brain to an engine signature still in review, and leave the derive reading the working directory whenever an override is set.

Decision Record impact: `none`. The change removes one use site of `aiConfig.projectRoot` and changes no leaf, in line with ADR-0019 §5.
Structure map: N/A. No file is added or moved (`ai/services/github-workflow` and `ai/scripts/maintenance` stay as they are).

## Related

Parent: #459 · `neomjs/neo#19408` · `neomjs/neo#19322` · `neomjs/pages#14` · #298

Sweeps:
- Live latest-open: the latest 20 open issues here at 2026-10-05T13:18Z (#700 to #885). No equivalent.
- A2A: the last 30 inbox messages at 13:18Z. No claim on this surface; the only adjacent open PR, #887, touches `ai/mcp/client/config.mjs` alone.
- MC sweep: two queries ("SyncService rebuildContentIndexesAndSeo Portal indexes SEO derive Brain sync emit-only data sync pipeline resources/content"; "full manual mode derives Portal artifacts pushes issues back corpus-only publish-corpus github-content-sync emitGeneratedContentAndDerive"), 12 results. No prior decision; the only hit on this surface is Grace's 2026-10-05 #19408 turn, which mapped this row onto #459.
- Own-assignment sweep: 16 open in this repo, none overlapping.

Origin Session ID: 5267f5db-e1d4-4297-8570-4981234133aa

Retrieval Hint: "SyncService Portal derive retired rebuildContentIndexesAndSeo corpusRoot neo.mjs pin move"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-05T13:18:40Z @neo-opus-ada added the `enhancement` label
- 2026-10-05T13:18:40Z @neo-opus-ada added the `ai` label
- 2026-10-05T13:18:41Z @neo-opus-ada added the `refactoring` label
- 2026-10-05T13:18:41Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-05T13:18:41Z @neo-opus-ada added the `agent-os` label
- 2026-10-05T13:18:48Z @neo-opus-ada added parent issue #459
- 2026-10-05T13:29:54Z @neo-opus-grace cross-referenced by PR #19408
- 2026-10-05T13:35:59Z @neo-opus-ada cross-referenced by PR #889
- 2026-10-05T13:58:56Z @tobiu referenced in commit `a38f232` - "fix(sync): the GitHub Workflow sync stops deriving Portal indexes and SEO (#888) (#889)

The derive step, `SyncService.rebuildContentIndexesAndSeo`, ran only in the manual modes
(`runFullSync` and `--emit-only`). The one scheduled caller, github-content-sync's
publish-corpus, runs `--corpus-only`, which never derived. The Portal's deploy now derives
from the published corpus (neomjs/pages#14). The step also called the engine with `{root}`
alone, so moving the pin past neomjs/neo#19408 would have made it throw.

- SyncService: `rebuildContentIndexesAndSeo` and the `deriveContent` option are removed, and
  `emitGeneratedContentAndDerive` becomes `emitGeneratedContent`. The auto-push allowlist
  drops the three `apps/portal/*` paths that only the derive wrote.
- syncGithubWorkflow.mjs: the docblock names the real scheduled caller and its mode.
- Specs: the derive arms are removed. Stage2 counts passes with `pullRuns`, and its
  allowlist arm asserts that no Portal path remains. RechunkOrdering keeps its property,
  re-anchored on the aggregate verdict. Its provenance paragraph moves here: the spec is
  half B of arm #3 of the Engine's RebuildContentIndexesAndSeo.spec.mjs (deleted by
  neomjs/neo@c623b2f63c), whose half A stays Engine-side under neomjs/neo#17922 (#298)."
- 2026-10-05T13:58:56Z @tobiu closed this issue

