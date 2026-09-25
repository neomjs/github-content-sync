---
id: 490
title: The Knowledge Base cannot read the corpus's release notes
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T14:13:55Z'
updatedAt: '2026-09-25T15:25:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/490'
author: neo-opus-grace
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
  - '[x] 485 Corpus mode emits no release notes, so corpus-only readers have none'
blocking: []
closedAt: '2026-09-25T15:25:51Z'
---
# The Knowledge Base cannot read the corpus's release notes

> **Amended 2026-09-25 by the author, at implementation.** The Fix moves the notes into a second extractor rather than extending `ConversationCorpusSource`, following the KB's model of one extractor per content family. It is not for failure isolation: the profile runner awaits each route with no catch, so a refusing route aborts the run either way. It also adds the exact-version ranking. The shard index's per-item `path` lands in #491.

## Context

#485 makes the published corpus carry each origin's release notes under `<origin>/release-notes/chunk-N/v*.md`, each set with its own shard index at `<origin>/release-notes/_index.json`. The Knowledge Base ingests the corpus as its own tenant through `ConversationCorpusSource`. This ticket is the consumer half, which #485 split off at intake because it sits across a separate service boundary.

## The Problem

- `ConversationCorpusSource` reads the corpus only through the root `_index.json`, and it knows three facets: `issues`, `pulls` and `discussions` (`FACETS`, `ai/services/knowledge-base/source/ConversationCorpusSource.mjs:28-32`). A note under `<origin>/release-notes/` yields nothing.
- The root manifest can't carry the notes either. `loadIndex` refuses the whole invocation for any row with an unknown type or a non-integer id ("needs a known type, a positive integer id and a path"), and a release is keyed by its tag.
- The KB types `release` today from engine paths (`resources/content/release-notes/`, `.github/RELEASE_NOTES/`). Those paths leave with the mirror in neomjs/neo#19159, and after that the KB has no release notes at all. `ReleaseNotesSource` does not fill the gap: it reads only top-level `.github/RELEASE_NOTES/*.md`, which neither repository has (neomjs/neo#19051, fact 7).

## The Architectural Reality

- The corpus tenant reads through a route-bound revision reader. A path its route does not declare is refused with `KB_REVISION_READER_PATH_OUTSIDE_SCOPE`.
- `ReleaseNotesSyncer.syncNotes` writes the shard index as `{metadata: {shardType: 'release-notes', …}, items: {<tag>: {itemIndex, chunk, chunkDir, path}}}`, where `path` is added in #491. A note's file name is prefixed (`v13.1.0.md` for the bare tag `13.1.0`), so without `path` a reader would have to re-derive the producer's prefix rule.
- `QueryService` filters vector results on the chunk's stored `type` (`where: {type}`). Its exact-version boost (`releaseExactMatch`) compares the whole name to the query. `inferSourceType`'s path prefixes serve only the lexical rescue of local files, never tenant chunks.

## The Fix

- A second corpus extractor, `ReleaseNotesCorpusSource`:
  - It emits one `release` chunk per note, named `<origin>/<file>` (`neo/v13.1.0`), with the origin and tag in `customMeta`.
  - A note's identity is the tag of the item whose `path` names it. A file the index does not name yields nothing.
  - An index that cannot say which note is which refuses, as the root index does.
  - It takes the same `origins` option as the conversation route, through one shared normalizer.
- The corpus tenant route in `deploy/cloud/kb-config.yaml` gains the second route, declaring `*/release-notes/_index.json` and `*/release-notes/**/*.md`. The root manifest's contract does not change.
- `QueryService`'s exact-version boost also matches a name that ends in `/<query>`. So a query for `v13.2.0` still ranks its note first once the engine path is gone.

## Acceptance Criteria

- [ ] A corpus fixture with `neo/release-notes/` yields `release` chunks that carry `neo` as their origin. An origin without notes yields none, with no refusal.
- [ ] A `type: 'release'` query reaches Chroma as `where: {type: 'release'}`, and an exact version ranks its corpus note above a note whose file name merely contains it.
- [ ] Post-merge, flagged: after the first corpus revision with notes, the deployed tenant answers a `release` query.

## Out of Scope

- Emitting the notes (#485).
- `ReleaseNotesSource`'s engine path, which retires with neomjs/neo#19159.

## Related

#485 (blocked-by) · #491 · #402 (the corpus tenant) · neomjs/neo#19157 · neomjs/neo#19159 · neomjs/neo#19051

Live latest-open sweep: the latest 20 open `neomjs/neo-agent-brain` issues at 2026-09-25T14:13:00Z, plus `release notes knowledge base corpus` and `ReleaseNotesSource` across Brain issues. No equivalent. #459 (Brain readers falling back to `resources/content`) is related but covers other readers.
A2A in-flight sweep: no claim on KB release-note ingestion in the last 30 messages.
MC sweep: `query_raw_memories("knowledge base does not ingest release notes from the github-content-sync corpus tenant; ConversationCorpusSource three facets; ReleaseNotesSource reads .github/RELEASE_NOTES zero notes")`, 5 results, no prior decision.
Own-assignment sweep: #485 is the producer half. Nothing else is on this surface.

Decision Record impact: none

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201

Authored by 🖖 **Grace** · `@neo-opus-grace` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-09-25T14:13:56Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T14:13:56Z @neo-opus-grace added the `enhancement` label
- 2026-09-25T14:13:57Z @neo-opus-grace added the `ai` label
- 2026-09-25T14:14:02Z @neo-opus-grace marked this issue as being blocked by #485
- 2026-09-25T14:14:26Z @neo-opus-grace cross-referenced by #485
- 2026-09-25T14:19:11Z @neo-opus-grace cross-referenced by PR #491
- 2026-09-25T14:54:58Z @neo-opus-grace referenced in commit `103df18` - "feat(github-workflow): the release-notes index names each note's path (#485)

Each item now carries the note's content-root-relative path, the form the corpus root index uses. A reader such as #490 takes a note's identity from the index instead of re-deriving the producer's filename prefix: neo's tags are bare (13.1.0), while its files are v13.1.0.md."
- 2026-09-25T15:05:16Z @neo-opus-grace cross-referenced by PR #494
- 2026-09-25T15:16:54Z @neo-opus-grace cross-referenced by #19157
- 2026-09-25T15:25:51Z @tobiu referenced in commit `94fe224` - "Merge pull request #494 from neomjs/grace/490-kb-corpus-release-notes

feat(knowledge-base): the corpus tenant reads each origin's release notes (#490)"
- 2026-09-25T15:25:52Z @tobiu closed this issue

