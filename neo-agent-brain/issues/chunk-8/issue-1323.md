---
id: 1323
title: Corpus emission can force-refetch named pulls and discussions
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-27T21:18:09Z'
updatedAt: '2026-10-27T22:08:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1323'
author: neo-opus-ada
commentsCount: 0
parentIssue: 1174
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-27T22:08:49Z'
---
# Corpus emission can force-refetch named pulls and discussions

## Context

This is Leaf A of #1174's corpus leg (Mnemosyne's lane-open, 2026-10-19). The corpus has to show renamed authors under their current logins. GitHub re-attributes an item when it is fetched, but the emitter only syncs deltas. #1049 (PR #1050) gave the issues facet a heal for named numbers, `--refetch-issues`. Pulls and discussions have no equivalent.

The issue facet is done. On 2026-10-27, 13 dispatched publisher runs (`refetch_issues`, every origin) rewrote each issue file whose attribution named a former login, and archived files were rewritten in place. Measured at `neomjs/github-content-sync@9e6bc469`:

| facet | files whose attribution names a former login |
|---|---|
| issues (every origin, archive included) | 0 (two mentions remain in body text, which GitHub never rewrites) |
| pulls | 3,317 (neo archive 3,059 · brain 160 · institution 72 · skills 21 · devindex 5) |
| discussions | 230 (neo 126 · neo archive 104) |

Instrument: a whole-token match of the former logins against frontmatter `author:` and the renderers' attribution headings (`` ### `@x` commented on ``, `` ### `@x` (STATE) reviewed on ``, `` #### Reply depth=N by `@x` ``, `### @x - `).

## The Problem

The content-sync repository owns publication (its README, design decision 2). Its `publish-corpus.yml` job runs only the emitter, `npm run ai:sync-github-workflow -- --corpus-only`, and the emitter parses no refetch flag except `--refetch-issues` (`ai/scripts/maintenance/syncGithubWorkflow.mjs`, `parseRefetchIssues`). The pull and discussion refetch primitives exist and have unit tests, but they are called only from out-of-band scripts (`ai/scripts/migrations/refetchStalePulls.mjs`, `refetchStaleDiscussions.mjs`) against the seat's own content root, and nothing publishes that root. So when a published PR or discussion file drifts without an `updatedAt` change, nothing can heal it: the delta skips it, and a dispatch cannot name it.

## The Architectural Reality

- `SyncService.emitConversationCorpus({refetchIssues})` → `emitGeneratedContent` → facet `issues` → `IssueSyncer.pullFromGitHub(metadata, {refetchIssues})`. Named numbers join the force-update set even when the delta already returned them, and the pull throws when one does not come back.
- Facet `discussions` → `DiscussionSyncer.syncDiscussions(metadata)`. Facet `pulls` → reconcile → `PullRequestSyncer.syncPullRequests(metadata)` → `repairDuplicateArtifacts` → `reconcilePullRequestIndex` → `verifyCorpusIntegrity`. When the integrity verdict fails, the facet does not advance.
- `PullRequestSyncer.refetchPullsByNumber(numbers, metadata, indexMutations)` and `DiscussionSyncer.refetchDiscussionsByNumber(…)` fetch one item each, render it with the bulk renderer, and write its metadata row. Placement differs between the two:
  - A terminal PR with one archived copy is rewritten at that path (`#getPullRequestPath`).
  - A discussion's bucket is recomputed from `closedAt` and the release list. At `9e6bc469`, that bucket is the folder every archived discussion already lives in.
- **Containment gap:** `syncDiscussions` quarantines entries of `issueSync.discussionDenylist` (`#isDenylisted`), but `refetchDiscussionsByNumber` never consults the list, and `#getDiscussionPath` has no gate. A named denylisted discussion would therefore be written. The issue path is safe: `#getIssuePath` returns `null` for a denylisted issue, so the issue is not written, and naming it fails the pull. The gap is latent today because the default denylist is empty, and the publisher cannot reach it until this ticket wires the path in.
- **Copy gap** (found while implementing): once the delta stops visiting a closed discussion, no pass ever archives it.
  - At `9e6bc469`, 58 of the 350 neo discussion files are closed but still sit in the active tier. The planner would bucket them into `v13.2.0` (49) and `v13.3.0` (9).
  - `refetchDiscussionsByNumber` writes the planned path and leaves the old file where it is. The emission's collision check scans only the archive, so the run would publish both copies.
  - The issue path deletes its stale active copies (`IssueSyncer#removeOtherCopies`). The pull facet archives closed PRs before it syncs.

Owning folders (structure map): `ai/services/github-workflow/sync/`, `ai/services/github-workflow/SyncService.mjs`, `ai/scripts/maintenance/syncGithubWorkflow.mjs`. No new files, and no config keys.

## The Fix

1. `syncGithubWorkflow.mjs`: add `--refetch-pulls=<n,…>` and `--refetch-discussions=<n,…>` under the `--refetch-issues` rules (one flag each, positive integers only, refused before any request). Pass both to `emitConversationCorpus`.
2. `SyncService`: `emitConversationCorpus` and `emitGeneratedContent` take `refetchPulls` and `refetchDiscussions` and hand each list to its facet.
3. `PullRequestSyncer.syncPullRequests(metadata, {refetchPulls})` and `DiscussionSyncer.syncDiscussions(metadata, {refetchDiscussions})`: after the delta, refetch the named items through the existing primitive, inside the same index update. A named item that does not come back throws, so its facet does not advance and the run publishes nothing. The pull facet's duplicate repair, index realign and integrity verdict then also cover the rewritten files.
4. `refetchDiscussionsByNumber` never writes a denylisted discussion. It removes the discussion's index entry instead, as the issue path does for a denylisted issue.
5. `refetchDiscussionsByNumber` leaves its rendering as the discussion's only active copy, as `IssueSyncer#removeOtherCopies` does for issues.

## Acceptance Criteria

- [ ] `--corpus-only --refetch-pulls=<n,…> --refetch-discussions=<n,…>` rewrites exactly the named items of the origin, and rewrites archived ones in place. A malformed or repeated flag is refused before any request.
- [ ] A named PR or discussion that does not come back fails its facet, and the emitter exits nonzero.
- [ ] The pull facet's integrity verdict runs after the named refetch.
- [ ] The refetch path never writes a denylisted discussion: the spec is red on `dev` and green at head.
- [ ] A refetched discussion leaves one copy. A closed discussion still in the active tier moves into its release bucket: the spec is red on `dev` and green at head.
- [ ] An emission without the new flags behaves as before: the existing emitter and syncer specs stay green.

## Out of Scope

- The publisher's dispatch inputs and runtime guard. They belong to a `neomjs/github-content-sync` ticket, as `neomjs/github-content-sync#22` did for #1049.
- Running the pull and discussion refetch itself, which finishes #1174's AC 3.
- Body text, and the comment and review caps the bulk renderer applies.

## Avoided Traps

- **An author-keyed refetch inside the emitter** (the wording of the 10-19 Leaf A). GitHub cannot search by a login that no longer exists, so the author set has to come from the corpus text in any case. Putting that census inside the emitter would ship a one-off rename migration in the publisher's runtime. Named numbers are the general heal (deleted comments, truncated timelines, renames). They reuse the tested primitives and the existing dispatch shape, and they keep the census outside the emitter, where the issue campaign already ran it.
- **Publishing the out-of-band scripts' output by hand.** Publication belongs to the content-sync job, and a corpus file and its `_index.json` row move together.

## Decision Record impact

none.

## Related

#1174 (parent outcome), #1049 / PR #1050 (issue precedent), `neomjs/github-content-sync#22`, #1190.

Not a `good first issue`: it is owned, because it unblocks the #1174 refetch that is already in progress.

Sweeps:
- Live latest-open: the 20 latest open Brain issues and every open content-sync issue, at 21:15Z. No equivalent.
- A2A claims: the last 30 messages. None overlap.
- Memory Core: `query_raw_memories` on stale corpus authors found the 10-19 lane-open, with no leaf filed.
- Own assignments: no work on the same surface.

Origin Session ID: 22826da5-42b6-4d60-b8fd-103b1ea0d925
Retrieval Hint: "corpus emission force-refetch named pulls discussions publisher containment denylist"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-10-27T21:18:09Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-27T21:18:11Z @neo-opus-ada added the `enhancement` label
- 2026-10-27T21:18:11Z @neo-opus-ada added the `ai` label
- 2026-10-27T21:18:12Z @neo-opus-ada added the `agent-os` label
- 2026-10-27T21:18:18Z @neo-opus-ada added parent issue #1174
- 2026-10-27T21:37:34Z @neo-opus-ada cross-referenced by #24
- 2026-10-27T21:38:43Z @neo-opus-ada cross-referenced by PR #1324
- 2026-10-27T22:08:49Z @tobiu referenced in commit `40bb74c` - "feat(github-workflow): corpus emission force-refetches named pulls and discussions (#1323) (#1324)

* feat(github-workflow): corpus emission force-refetches named pulls and discussions (#1323)

The emitter healed named issues only (--refetch-issues, #1049). Pulls and
discussions gain --refetch-pulls and --refetch-discussions under the same
rules: parsed before the emitter loads, refetched inside their facet's own
index update, and a named item that does not come back fails the facet.

refetchDiscussionsByNumber now honors the discussion denylist, so a
recovery run cannot write back what the bulk sync quarantines. The gate
reads its leaf without a fallback (ADR 0019 B3), like its issue sibling.

* fix(github-workflow): a discussion refetch leaves its rendering as the only active copy (#1323)"
- 2026-10-27T22:08:49Z @tobiu closed this issue

