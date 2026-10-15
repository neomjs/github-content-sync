---
id: 1038
title: The corpus keeps stale issue renderings over fresh ones
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-15T09:20:09Z'
updatedAt: '2026-10-15T12:38:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1038'
author: neo-opus-ada
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
closedAt: '2026-10-15T11:18:21Z'
---
# The corpus keeps stale issue renderings over fresh ones

## Context

The neomjs/neo#19712 release-notes scope run on 2026-10-15 counted 24 closed `neomjs/neo` issues since 13.2.0 in the github-content-sync corpus (`729c0e8d`, published 07:25Z). GitHub has 50. Every one of the 26 missing issues has a corpus file that still reads `state: OPEN`, with `updatedAt` frozen before its close. One of them, neo#19739, closed after the publication. The other 25 are stale: neo#19565, #19637, #19640, #19643, #19656–#19658, #19661–#19666, #19670, #19671, #19684, #19685, #19687, #19692–#19694, #19697, #19698, #19720 and #19725. One example: `neo/issues/chunk-9/issue-19666.md` still carries its 10-12T14:10:00Z title ("a 100k wrapped-text example…"). On GitHub that issue was renamed and closed on 10-13. A defect-note went out at 08:57Z.

## The Problem

The publisher run [38319569023](https://github.com/neomjs/github-content-sync/actions/runs/38319569023) (2026-10-12 22:57Z) shows the loss. The issue facet found 27 `neomjs/neo` issues updated since the last sync and wrote them. Then the active-tier reconcile logged, for neo#19643, #19655–#19658 and #19661–#19666:

```
[WARN] [reconcileActiveChunks] removed duplicate issue-19662.md at …/neo/issues/chunk-3/issue-19662.md
```

The removed copies were the fresh renderings. That run removed 12 issue duplicates and 1 discussion duplicate; run 38342193879 (10-13 03:32Z) removed 16 more.

Nothing repairs the loss. The delta sync (`since: metadata.lastSync`) refetches an issue only when GitHub updates it again. The metadata already holds the fresh `updatedAt`, so `needsUpdate` stays false. The stale file is permanent, and the Knowledge Base's `github-content-sync` tenant serves these tickets to every agent as open, with old bodies.

## The Architectural Reality

Three code paths combine:

1. **`IssueSyncer.pullFromGitHub` writes, then renames the old file onto what it wrote.** When an issue's planned path moves, the syncer writes the fresh markdown to `targetPath` and then calls `fs.rename(oldAbsolutePath, targetPath)`. POSIX rename replaces the destination, so the old content overwrites the new. This has been unchanged since the import (`11552b08`). `DiscussionSyncer` does it right: it writes, then unlinks the old path.
2. **The old path comes from metadata, which the reconcile does not update.** `oldAbsolutePath` resolves `metadata.issues[n].path`. `reconcileActiveChunks` moves files to their ordinal chunk and realigns `_index.json`, but never touches the sync metadata. On the next write the metadata path no longer names the file. The same run logs `Could not rename #6858, falling back to write … ENOENT: rename chunk-1/issue-6858.md -> chunk-2/issue-6858.md`. A write at a path the file has left creates a second copy beside the real one. `pullFromGitHub` already holds `buildContentInventory`'s complete on-disk map of every copy per id; it passes the map to `#planBuckets` and never uses it to find the old file.
3. **The dedup keeps whichever copy sorts first by path.** `reconcileActiveChunks` sorts on `(id, absPath.localeCompare)` and unlinks every later copy, so lexical order picks the survivor: `chunk-1x` sorts before `chunk-3`. `PullRequestSyncer#repairDuplicateArtifacts` (#623) says keep-first "is safe in the active tier only because the next sync rewrites the survivor from GitHub". The delta sync does not rewrite an unchanged issue. PRs get away with it because they are repaired from GitHub before the reconcile runs; issues and discussions are not.

## The Fix

- **`IssueSyncer` (bulk pull and `refetchIssuesByNumber`):** after writing the fresh rendering to `targetPath`, unlink every other active-tier copy of that id: the metadata path plus the inventory's copies. Never rename an old copy onto the target. This removes both the clobber (1) and the source of duplicates (2).
- **`reconcileActiveChunks`:** among the copies of one id, keep the one whose frontmatter `updatedAt` is newest; a tie keeps the first. GitHub's `updatedAt` never decreases, so a stale copy cannot win. Only duplicate groups are read, so a clean tier costs nothing extra.
- **Remediation:** a one-time refetch of the stale numbers through `SyncService#refetchIssuesByNumber` on the publisher, once the fixed pin is live. This is post-merge validation, not part of the PR.

Decision Record impact: none. The change is aligned with ADR 0004's ordinal-100 chunking and sealed archive, and touches no archive tier.

## Acceptance Criteria

- [x] AC-1: In `IssueSyncer`, an updated issue whose planned path moves ends with exactly one file, at `targetPath`, holding the fresh rendering. Unit test, red on `dev`.
- [x] AC-2: A sync write for an issue whose metadata path is stale, because the file was moved, leaves no second active-tier copy. Unit test with a temporary content root, red on `dev`.
- [x] AC-3: Given two copies of one id where the lexically first copy holds the older `updatedAt`, `reconcileActiveChunks` keeps the newer copy, for each content type. Unit test, red on `dev`.
- [x] AC-4 (post-merge): after the publisher runs on the fixed pin and the stale numbers are refetched, no `neomjs/neo` issue that GitHub has closed is `OPEN` in the corpus. Check: corpus `state: OPEN` set minus `gh issue list --state open`. Done 2026-10-15: the publisher dispatch (neomjs/github-content-sync#22 / #1050) published `2ee9271`, and the check returned none at 12:36Z. Readback: https://github.com/neomjs/github-content-sync/issues/3#issuecomment-6192203997

## Out of Scope

- 103 issues that are open on GitHub have no `OPEN` file in the corpus's active tier. This is a separate observation (caps, labels or the archive tier) and has not been investigated.
- PR duplicate repair (#623), which works as designed.
- The `github-content-sync` workflow itself, beyond the post-merge refetch.

## Avoided Traps

- **Refetching duplicates after dedup instead of fixing the writer.** That leaves the clobber in (1), which needs no duplicate at all.
- **Trusting `metadata.path` for the old location.** The inventory exists because the metadata drifts (`contentInventory.mjs`'s own rationale).

## Related

#387 / #388 (emitter) · #623 (PR duplicate repair) · neomjs/github-content-sync#3 · neomjs/neo#19712

Sweeps at 2026-10-15T09:17Z: the latest 20 open neo-agent-brain issues hold no equivalent. Keyword searches (`reconcileActiveChunks`, stale or duplicate corpus) found no open ticket. My open assignments here are not on this surface. The A2A channel since 08:00Z carries only my defect-note. Placement: existing files in `ai/services/github-workflow/{sync,shared}`, so no new placement.

Retrieval Hint: "reconcileActiveChunks removed duplicate fresh copy", "IssueSyncer rename old onto target", "corpus closed issues stale OPEN"


## Timeline

- 2026-10-15T09:20:10Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-15T09:20:10Z @neo-opus-ada added the `bug` label
- 2026-10-15T09:20:10Z @neo-opus-ada added the `ai` label
- 2026-10-15T09:32:08Z @neo-opus-ada cross-referenced by PR #1040
- 2026-10-15T10:12:44Z @neo-opus-ada referenced in commit `f668a79` - "fix(github-workflow): stale-copy cleanup compares resolved paths, so it never deletes the fresh write (#1038)

#resolvePath passes an absolute metadata path through unnormalized, so a spelling such as chunk-1/../chunk-1/issue-N.md escaped the Set exclusion of the target and unlinked the rendering just written. #removeOtherCopies now resolves the candidates, the target and the archive root before comparing. Two regressions, through the bulk pull and through refetchIssuesByNumber, failed first and now keep exactly the fresh file."
### @neo-opus-ada - 2026-10-15T10:37:55Z

## Handover (session sunset, solo-refresh)

PR #1040 is in Round 2. Euclid's one Required Action, which was to compare resolved paths so the cleanup never deletes the fresh write, is addressed in `f668a79b`, with two `..` regressions through the bulk and refetch paths. They were red first and are green now. Its `unit` job reddened once on the unrelated #1042 marker flake. I re-ran the job, and #1045, merged since, retires that flake.

**AC-4 after merge (post-merge, Residual-Owner neomjs/github-content-sync#3):**
1. The publisher runs Brain `dev` unpinned, so its next scheduled run uses the fixed syncer.
2. The 25 stale `neomjs/neo` issues listed in this body still need one refetch through `SyncService#refetchIssuesByNumber`. The publish workflow has no refetch input; agree the route with that ticket's owner, either a dispatch input or a one-time run.
3. The check: corpus `state: OPEN` set minus `gh issue list --state open`, sorted lexically for `comm`, returns no closed issue.

Not investigated, and separate: 103 issues open on GitHub have no `OPEN` file in the corpus's active tier.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-10-15T11:00:44Z @neo-opus-ada referenced in commit `d8b2dc7` - "chore(merge): bring dev's #1045 deadline-refusal fix into the corpus branch (#1038)"
- 2026-10-15T11:18:21Z @tobiu referenced in commit `6cd40fe` - "fix(github-workflow): the issue corpus keeps the fresh rendering over stale copies (#1038) (#1040)

* fix(github-workflow): the issue corpus keeps the fresh rendering over stale copies (#1038)

IssueSyncer wrote the new markdown to targetPath and then renamed the old file onto it, so an updated issue whose planned path moved kept its old content. It also found the old file only through the metadata path, which reconcileActiveChunks never updates when it relocates a file, so a write could land beside the real copy. The syncer and refetchIssuesByNumber now unlink every other active-tier copy the inventory holds, plus the metadata path, after writing. reconcileActiveChunks' dedup keeps the copy with the newest frontmatter updatedAt instead of the first in path order.

* fix(github-workflow): stale-copy cleanup compares resolved paths, so it never deletes the fresh write (#1038)

#resolvePath passes an absolute metadata path through unnormalized, so a spelling such as chunk-1/../chunk-1/issue-N.md escaped the Set exclusion of the target and unlinked the rendering just written. #removeOtherCopies now resolves the candidates, the target and the archive root before comparing. Two regressions, through the bulk pull and through refetchIssuesByNumber, failed first and now keep exactly the fresh file."
- 2026-10-15T11:18:21Z @tobiu closed this issue
- 2026-10-15T11:28:33Z @neo-opus-ada cross-referenced by #1049
- 2026-10-15T11:28:36Z @neo-opus-ada cross-referenced by #22
- 2026-10-15T11:37:45Z @neo-opus-ada cross-referenced by PR #1050
- 2026-10-15T11:40:59Z @neo-opus-ada cross-referenced by PR #23
- 2026-10-15T12:08:40Z @neo-opus-ada cross-referenced by #3

