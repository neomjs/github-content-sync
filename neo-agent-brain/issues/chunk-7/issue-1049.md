---
id: 1049
title: Corpus emission can force-refetch named issues
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-15T11:28:32Z'
updatedAt: '2026-10-15T12:29:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1049'
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
closedAt: '2026-10-15T12:29:36Z'
---
# Corpus emission can force-refetch named issues

## Context

#1040 merged on 2026-10-15 and closed #1038. Its post-merge AC-4 is still open. 25 `neomjs/neo` issues that GitHub has closed still read `OPEN` in the github-content-sync corpus. #1038 lists them, and its check returned exactly those 25 against the corpus at `729c0e8` (07:25Z), plus two that only lag. The delta sync (`since: metadata.lastSync`) never refetches them, because their metadata already holds the fresh `updatedAt`. #1040 stops new clobbers; it cannot heal these.

Vega owns the publisher (neomjs/github-content-sync#3). On 2026-10-15 at 11:22Z they chose a dispatch input on `publish-corpus.yml` over a hand-made corpus PR or label churn, and asked me to author it with two conditions:
- the numbers are validated as integers before anything runs;
- the refetched files go through the same admission and commit step as an ordinary run.

This ticket is the Brain half: the runtime entry the publisher invokes has to accept the numbers.

## The Problem

The heal primitive exists, `SyncService#refetchIssuesByNumber` (CLI: `ai/scripts/migrations/refetchTruncatedIssues.mjs`), but the publisher cannot use it as it stands:
- The script imports `ai/services.host.mjs`. That graph includes Neural Link's `ConnectionService`, which connects on init. On a runner, that risks a Bridge spawn or a step that never exits.
- It skips `assertCorpusDestination()` and the corpus lease.
- `SyncService#refetchIssuesByNumber` passes no `indexMutations`, so a refetch that moves a file leaves `_index.json` behind.

## The Architectural Reality

- `ai/scripts/maintenance/syncGithubWorkflow.mjs --corpus-only` is the publisher's entry point. By design it imports `SyncService` directly, not through the barrel. It validates the destination before loading the emitter, and runs under the corpus lease.
- `IssueSyncer#pullFromGitHub` already force-refetches a set, the related issues found in fetched timelines. It goes through `refetchIssuesByNumber(…, newMetadata, indexMutations)` and then `updateContentIndex()`, inside the `issues` facet, whose metadata `SyncService#runFacet` rolls back on failure.
- `refetchIssuesByNumber` warns and skips an issue GitHub does not return, so a mistyped number would pass silently.

## The Fix

- `syncGithubWorkflow.mjs` gains `--refetch-issues=<n,n,…>`. It is accepted only with `--corpus-only`. Every token must be a positive integer, checked before the emitter loads; otherwise the CLI exits 1 and names the token.
- `SyncService#emitConversationCorpus({refetchIssues})` passes the numbers through `emitGeneratedContent` to `IssueSyncer#pullFromGitHub(metadata, {refetchIssues})`. They seed its force-refetch set, and the existing dedup drops any number the delta already pulled.
- A named number that is not refetched fails the `issues` facet with its number. An explicit request fails closed.

## Acceptance Criteria

- [ ] AC-1: `--refetch-issues` with a token that is not a positive integer, or without `--corpus-only`, exits 1 before the emitter loads, naming the problem.
- [ ] AC-2: The actual corpus CLI with `--refetch-issues` rewrites a stale copy of a named issue whose metadata already holds the fresh `updatedAt`, and `_index.json` and `.sync-metadata.json` carry its entry.
- [ ] AC-3: A named number that cannot be refetched makes the run exit 1, naming the number, and the `issues` facet does not advance.
- [ ] AC-4: The CLI's header documents the flag.

## Out of Scope

- The publisher's dispatch input and its validation: neomjs/github-content-sync#22.
- Refetch flags for pulls and discussions.
- Running the remediation: post-merge, on the publisher.

## Related

#1038 · #1040 · neomjs/github-content-sync#3 · neomjs/github-content-sync#22

Decision Record impact: none. Structure map: three existing owning files (`ai/scripts/maintenance`, `ai/services/github-workflow`, `ai/services/github-workflow/sync`); no new placement.

Sweeps at 2026-10-15 11:28Z: the latest 20 open issues here and github-content-sync's one open issue (#3) hold no refetch route. An org search for "refetch" finds #65 (mirror ingestion) and neomjs/neo#17416, neither of them this. My open assignments here are not on this surface. A2A since 11:00Z holds no competing claim. Memory Core holds no earlier decision on a corpus refetch route.

Retrieval Hint: "corpus refetch named issues", "refetch-issues corpus-only", "publisher heal stale corpus files"


## Timeline

- 2026-10-15T11:28:33Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-15T11:28:33Z @neo-opus-ada added the `enhancement` label
- 2026-10-15T11:28:33Z @neo-opus-ada added the `ai` label
- 2026-10-15T11:29:04Z @neo-opus-ada cross-referenced by #22
- 2026-10-15T11:37:45Z @neo-opus-ada cross-referenced by PR #1050
- 2026-10-15T12:06:37Z @neo-gpt-emmy cross-referenced by PR #23
- 2026-10-15T12:29:36Z @tobiu referenced in commit `fa32648` - "feat(github-workflow): corpus emission force-refetches the issues it is named (#1049) (#1050)"
- 2026-10-15T12:29:37Z @tobiu closed this issue

