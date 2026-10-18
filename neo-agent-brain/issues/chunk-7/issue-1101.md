---
id: 1101
title: Retire the full and --emit-only GitHub sync modes and their delivery
state: OPEN
labels:
  - enhancement
  - ai
  - refactoring
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T03:51:58Z'
updatedAt: '2026-10-18T04:27:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1101'
author: neo-opus-grace
commentsCount: 0
parentIssue: 17416
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 459 Brain readers fall back to resources/content under projectRoot or __dirname'
blocking: []
---
# Retire the full and --emit-only GitHub sync modes and their delivery

## Context

The neomjs/neo#19158 census ([IC 5815306776](https://github.com/neomjs/neo/issues/19158#issuecomment-5815306776)) has one writer row: `SyncService.mjs:23-29,480`, `PrimaryRepoSyncService.mjs:8,50`, `migrate-discussions-b1.mjs:8`. It classes the row as "the in-engine sync path (`runFullSync` → `commitRebaseAndPushGeneratedContent`), the engine's sync metadata, a one-off migration", with the disposition "retire (#19159)".

neomjs/neo#19159 closed after deleting the engine's tree, and its AC-3 handed the Brain only the post-release lifecycle (#623). Nothing retired the rest. Every site is still on `dev` at `a9bd0338`, and no open issue or PR names them (sweeps below). This was found while reviewing #1100 ([5561310364](https://github.com/neomjs/neo-agent-brain/pull/1100#pullrequestreview-5561310364)).

## The Problem

Two CLI modes have no consumer left, and both still act:
- `npm run ai:sync-github-workflow` with no flag runs `runFullSync()`. That call:
  1. pushes local issue edits to the real GitHub issues (`UPDATE_ISSUE`, `IssueSyncer.mjs:1182`);
  2. emits the content;
  3. commits and pushes `resources/content/...` in whatever checkout it runs from.
- `--emit-only` does the same emission, without the push.

Today both modes write `<cwd>/resources/content`. In an engine checkout that re-creates the tree neomjs/neo#19159 deleted (neo `origin/dev` tracks 0 files there). After #1100 their root is `''`, so they read empty metadata, fetch the whole repository from 2017, and then throw (#1100's RA-3 makes that a refusal by name). In neither state does a mode produce anything a reader uses. The Portal derives from the published corpus (#888), and the publisher runs `--corpus-only` (neomjs/github-content-sync `publish-corpus.yml:208`).

The orchestrator carries the same dead weight. `PrimaryRepoSyncService` still treats `resources/content/` as a path that can affect the KB, and its pull ladder still resets `resources/content/.sync-metadata.json`, a file the engine no longer tracks.

## The Architectural Reality

Design authority: the census disposition — "the in-engine sync path (`runFullSync` → `commitRebaseAndPushGeneratedContent`), the engine's sync metadata, a one-off migration | retire (#19159)".

Line numbers below are at `a9bd0338`.

- `ai/scripts/maintenance/syncGithubWorkflow.mjs` (301 lines):
  - three modes (`:230-241`, `:266-270`); `assertSyncGithubWorkflowDevBranch` (`:114`, exported `:301`) wraps `syncGithubWorkflowBranchGuard.mjs` (`:43`);
  - the publisher greps this file for `corpus-only` and `--refetch-issues` before it invokes it (`publish-corpus.yml:161,168`).
- `ai/scripts/maintenance/syncGithubWorkflowBranchGuard.mjs` (72 lines): it keeps the full sync from writing a checkout that is not on `dev`. Its only consumers are the CLI and `syncGithubWorkflow.spec.mjs:12`.
- `ai/services/github-workflow/SyncService.mjs` (779 lines):
  - `generatedSyncPaths`, `isGeneratedSyncFile` and `generatedSyncStatusPaths` (`:24-38`);
  - `commitRebaseAndPushGeneratedContent` (`:490`), with a staged-set logical-identity check against `path.join(cwd, 'resources/content/archive')` (`:522-540`). The corpus path has asserted identity since #1093 (#457);
  - `recoverGeneratedContentCheckout`, `autoPushGeneratedContent` (`:585`) and `runFullSync` (`:628`), plus `execGit` and the `RepositoryService` import that only they use;
  - `emitGeneratedContent`'s `pushLocalChanges` option (`:181`). Only `runFullSync` leaves it `true`; `emitConversationCorpus` passes `false` (`:418`).
- `ai/services/github-workflow/sync/IssueSyncer.mjs`: `pushToGitHub` (`:1106`), its push-only helpers `#scanLocalFiles` and `#getPushPayloadFromMarkdown`, and the `GET_ISSUE_FOR_PUSH` / `UPDATE_ISSUE` import (`:13`). Neither query has another caller. The push's bookkeeping, `pushFailures`, runs through the pull, the issues facet and `MetadataManager.save`.
- `ai/mcp/server/github-workflow/configBase.mjs`: `pushToRepoAfterSync` is read only by the auto-push.
- `ai/daemons/orchestrator/services/PrimaryRepoSyncService.mjs`:
  - `resources/content/` sits in the KB-relevant prefixes (`:50`). The KB profile still routes `ConceptSource` at `resources/content/concepts` (`coreCorpusProfilePlan.mjs:47,56`), and the predicate mirrors the source registry;
  - `META_SYNC_PATH` (`:8`) feeds `KB_IRRELEVANT_PATHS` (`:60`), the ladder's reset step (`:186`, `:559-561`) and its predicate (`:943`).
- `ai/scripts/migrations/migrate-discussions-b1.mjs` (64 lines): a one-off that derives `resources/content/discussions` from `projectRoot` (`:8`). Nothing calls it.
- Docs that teach the full sync: `learn/agentos/GitHubWorkflow.md:73,93`, `learn/agentos/StrategicWorkflows.md:142`, and the CLI's JSDoc (`:52-54`, `:98-104`).
- Specs:
  - `syncGithubWorkflow.spec.mjs` (581 lines) covers the guard, the auto-push and the mode order (`:141`, `:267-275`);
  - `scriptPlaneClosure.spec.mjs:318` models the CLI's `runFullSync` spawn;
  - `postReleaseSync.spec.mjs:31` asserts `postReleaseSync` never calls `runFullSync`.

Structure map (`npm run ai:structure-map -- --files --loc`): `ai/scripts/maintenance` (51 files), `ai/services/github-workflow` (17), `ai/daemons/orchestrator/services` (44), `ai/scripts/migrations` (16). Every change is a deletion inside its owning folder, so nothing moves.

## The Fix

1. **CLI:** only `--corpus-only` (with `--refetch-issues`) remains. The bare command and `--emit-only` exit 1, before the lease or any request, and the message names `--corpus-only`. Both literals stay in the file.
2. **Guard:** delete `syncGithubWorkflowBranchGuard.mjs`, `assertSyncGithubWorkflowDevBranch` and their spec arms.
3. **`SyncService`:** delete `runFullSync`, `autoPushGeneratedContent`, `commitRebaseAndPushGeneratedContent`, `recoverGeneratedContentCheckout`, `execGit` and the three `generatedSync*` constants. `emitGeneratedContent` loses `pushLocalChanges`, and the issues facet loses its push half.
4. **`IssueSyncer`:** delete `pushToGitHub` and its push-only helpers. `GET_ISSUE_FOR_PUSH` and `UPDATE_ISSUE` leave `queries/mutations.mjs`. `pushFailures` leaves the pull, the issues facet and the saved metadata.
5. **Config:** retire the `pushToRepoAfterSync` leaf and its parity entry.
6. **`PrimaryRepoSyncService`:** narrow the `resources/content/` prefix to `resources/content/concepts/`, the root `ConceptSource` still reads. Drop `META_SYNC_PATH` together with its ladder step. The ladder becomes: fast-forward when the checkout is clean, otherwise skip with the warning.
7. **Migration:** delete `migrate-discussions-b1.mjs`.
8. **Docs and specs:** rewrite the two guides and the CLI JSDoc around the one mode, and retire spec assertions whose subject is gone (`postReleaseSync.spec.mjs:31`, the `scriptPlaneClosure` model).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `npm run ai:sync-github-workflow` (no flag) | census writer row → retire | exit 1 before the lease, naming `--corpus-only` | — | CLI JSDoc; `GitHubWorkflow.md`; `StrategicWorkflows.md` | CLI arm: no request, no lease |
| `--emit-only` | census writer row → retire | exit 1, same message | — | CLI JSDoc | CLI arm |
| `--corpus-only`, `--refetch-issues` | `publish-corpus.yml:161,168,208` | unchanged | — | unchanged | existing CLI arms pass unchanged |
| `SyncService#runFullSync`, `#autoPushGeneratedContent`, `#commitRebaseAndPushGeneratedContent`, `emitGeneratedContent({pushLocalChanges})`, `IssueSyncer#pushToGitHub` | census writer row | removed | — | `SyncService` JSDoc | AC-3 grep |
| `pushToRepoAfterSync` (github-workflow config leaf) | census writer row: the switch of the retired auto-push | removed with the auto-push, its only reader | — | — | `lint-config-template-ssot`; AC-3 grep |
| `<origin>/.sync-metadata.json` `pushFailures` key | the push's bookkeeping | dropped by the next save | — | — | `MetadataManager` spec |
| `PrimaryRepoSyncService` pull ladder | census ("the engine's sync metadata") | fast-forward when clean, else skip with the warning | — | class JSDoc | its spec |

**Decision Record impact:** none. No ADR in `learn/agentos/decisions/` names the full sync, `pushToGitHub` or `ai:sync-github-workflow` (searched at `a9bd0338`; control: 6 ADRs name `AiConfig`). This ticket executes the census disposition.

## Acceptance Criteria

- [ ] AC-1: the bare command and `--emit-only` exit 1 naming `--corpus-only`, before any lease or GitHub request (CLI arm).
- [ ] AC-2: `--corpus-only` and `--refetch-issues` behave as on `dev`. Their existing arms pass unchanged, and the file still contains both literals.
- [ ] AC-3: `git grep -nE "runFullSync|autoPushGeneratedContent|commitRebaseAndPushGeneratedContent|generatedSyncPaths|pushToGitHub|syncGithubWorkflowBranchGuard|pushLocalChanges|pushToRepoAfterSync|GET_ISSUE_FOR_PUSH|UPDATE_ISSUE|pushFailures" -- ai test` returns 0 hits, while the same command for `emitConversationCorpus` returns at least 1.
- [ ] AC-4: `PrimaryRepoSyncService.mjs` names no `resources/content` path other than `resources/content/concepts/`, the KB `ConceptSource` root (neomjs/neo#19093), and its spec passes.
- [ ] AC-5: `migrate-discussions-b1.mjs` is gone.
- [ ] AC-6: `GitHubWorkflow.md`, `StrategicWorkflows.md` and the CLI JSDoc describe `--corpus-only` as the only mode.

## Out of Scope

- The concept source paths (`resources/content/concepts` in the KB config and in `coreCorpusProfilePlan.mjs`): neomjs/neo#19093.
- `retireLegacyConversationRows.mjs`: it matches stored KB-row prefixes, not a filesystem path.
- The `refetch*` and `rebucketArchive` migrations: they read the declared leaves and run against a declared root.
- #1100's reader and formula changes.

## Avoided Traps

- **Making the bare command run corpus emission.** It would silently change what a remembered invocation does. A refusal names the change once.
- **Keeping `runFullSync` behind a declared root.** Its delivery stages `resources/content/...` in the repository it runs from, and its push rewrites real issues from local markdown. After #888 and neomjs/neo#19159, neither has a consumer.
- **Dropping the whole `resources/content/` prefix.** It would also stop a change under the concepts root from cascading a KB sync while `ConceptSource` still reads it.

## Related

neomjs/neo#17416 (parent epic) · neomjs/neo#19158 · neomjs/neo#19159 · #623 · #888 · #457 · #1093 · #459 · #1100. #1100 lands first, because both edit `emitGeneratedContent`.

Sweeps at 2026-10-18T03:50:48Z:
- **Live latest-open:** the latest 20 open issues here; none equivalent.
- **Search** (`runFullSync`, `emit-only`, `syncGithubWorkflowBranchGuard`, `generatedSyncPaths`, `migrate-discussions-b1`, `in-engine sync`): only closed neighbours (#623, #888, #457, #298).
- **PRs:** no open PR touches these files; none of mine does either.
- **A2A:** the last 30 messages, all read states; no claim on this scope.
- **MC:** "the ordinary GitHub sync still writes the engine's retired mirror; nobody retired runFullSync, the emit-only mode and the dev-branch guard". It returned two prior findings, that the full sync pushes local edits to real issues and that the publisher runs `--corpus-only`, and no decision against retiring the modes.
- **Own assignments:** #684, #18, #54; none overlaps.

Amended 2026-10-18 during implementation. Verification found more dead code than the census row named (the push's helpers, mutations and bookkeeping, plus the `pushToRepoAfterSync` leaf), and found `ConceptSource` still routed at `resources/content/concepts`, which narrows AC-4.

Assigned to me.

Origin Session ID: bab4a460-7586-4505-b324-56756d2694e5
Retrieval Hint: "retire runFullSync emit-only syncGithubWorkflow branch guard in-engine sync census writer row"

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-10-18T03:51:58Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T03:51:59Z @neo-opus-grace added the `enhancement` label
- 2026-10-18T03:52:00Z @neo-opus-grace added the `ai` label
- 2026-10-18T03:52:00Z @neo-opus-grace added the `refactoring` label
- 2026-10-18T03:52:00Z @neo-opus-grace added the `agent-os` label
- 2026-10-18T03:52:24Z @neo-opus-grace added parent issue #17416
- 2026-10-18T03:52:27Z @neo-opus-grace marked this issue as being blocked by #459

