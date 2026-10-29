---
id: 1353
title: ChromaManager.spec leaks knowledgeBaseCollectionBootstrapped into later specs
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-29T13:57:09Z'
updatedAt: '2026-10-29T13:57:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1353'
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
---
# ChromaManager.spec leaks knowledgeBaseCollectionBootstrapped into later specs

## Context

Dependabot's PR #1345 (`neo-agent-skills` 0.1.70 → 0.1.73, lock and manifest only) went red on `unit` with `1 introduced`. The introduced test is `KB_DatabaseService — manageDatabaseBackup (#10129 Phase 1)` › "returns gracefully without producing a JSONL when the collection is empty". It expected `reason: 'source-collection-empty'` and got `'source-collection-unresolved'` (run 40818612717). The skills bump doesn't cause it. It only reshuffled which specs share a worker.

## The Problem

`ChromaManager.knowledgeBaseCollectionBootstrapped` is a singleton flag (#270). The resolver writes it, and `DatabaseService.mjs:209` reads it when an export exports zero rows. A `true` there turns "empty" into "unresolved".

- **Writer that leaks.** `ChromaManager.spec.mjs` › "a CREATED canonical collection records `true`" leaves the flag `true`. The spec's `afterEach` restores `client`, `connected`, `_knowledgeBaseCollectionPromise`, `knowledgeBaseCollection` and `collectionResolveRetrySleepFn`, but not this flag. This is the asymmetric-restore shape #89 named.
- **Reader that inherits it.** The backup spec's empty arm stubs `getKnowledgeBaseCollection`, so the resolver never runs and the stale flag stands.

Measured 2026-10-29 on `dev@2ed12394` (skills 0.1.70), `--workers=1 --retries=0`:

| run | result |
|---|---|
| backup arm alone | passes |
| `ChromaManager.spec.mjs:131`, then the backup arm, in one worker | fails with exactly #1345's `source-collection-unresolved` |

## The Architectural Reality

The sibling `kbExportEmptyReceipt.spec.mjs` already handles this. Its `beforeEach` pins the flag with the comment: "`knowledgeBaseCollectionBootstrapped` describes whichever resolution ran last — including one performed by a different spec file". The backup spec's arm has no such pin. The census of test writers is `ChromaManager.spec` (leaks) and `kbExportEmptyReceipt.spec` (pins and resets). The census of readers is `DatabaseService` export, reached through `manageDatabaseBackup` by both specs.

## The Fix

1. `ChromaManager.spec.mjs`: capture the flag with the other originals and restore it in `afterEach`, so the writer stops leaking.
2. `DatabaseService.backup.spec.mjs`: the empty arm pins `knowledgeBaseCollectionBootstrapped = false`, since its stub stands in for a resolver that found the collection, and restores it afterwards. The arm then declares its own premise, like its sibling does.

## Acceptance Criteria

- [ ] AC-1: the one-worker pair (`ChromaManager.spec.mjs:131` then the backup empty arm) passes.
- [ ] AC-2: `ChromaManager.spec`'s `afterEach` restores `knowledgeBaseCollectionBootstrapped`.
- [ ] AC-3: the backup empty arm passes whatever value the flag holds when it starts.
- [ ] AC-4: Brain Unit on the PR reports 0 introduced reds.

## Out of Scope

- #1345 itself, which only needs a CI rerun or a Dependabot rebase once this lands.
- Moving the provenance flag off the singleton.

## Related

Parent: #201 (its shared-state/order-pollution class). Prior art: #89 (asymmetric restore), #270 (the flag). Surfaced by: #1345.

Live latest-open + closed sweep: latest 20 open Brain issues, plus a closed and open search for `knowledgeBaseCollectionBootstrapped` / `source-collection-unresolved` / `manageDatabaseBackup`, at 2026-10-29T13:56:49Z. The hits (#270, #66, #974) are prior art, and none is equivalent. A2A claim sweep: the unread queue at 13:55Z, with no claim. Own assignments: #201, #1346, #1347. Structure map: N/A, since this changes two specs.

Retrieval Hint: "knowledgeBaseCollectionBootstrapped singleton leak ChromaManager.spec afterEach source-collection-unresolved backup empty arm dependabot"

## Timeline

- 2026-10-29T13:57:09Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-29T13:57:10Z @neo-opus-ada added the `bug` label
- 2026-10-29T13:57:11Z @neo-opus-ada added the `ai` label
- 2026-10-29T13:57:11Z @neo-opus-ada added the `testing` label
- 2026-10-29T14:00:43Z @neo-opus-ada cross-referenced by PR #1354
- 2026-10-29T14:01:07Z @neo-opus-ada cross-referenced by PR #1345

