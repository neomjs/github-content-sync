---
id: 485
title: 'Corpus mode emits no release notes, so corpus-only readers have none'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-25T12:14:35Z'
updatedAt: '2026-09-25T14:14:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/485'
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
blockedBy: []
blocking:
  - '[ ] 490 The Knowledge Base cannot read the corpus''s release notes'
---
# Corpus mode emits no release notes, so corpus-only readers have none

## Context

@tobiu, 2026-09-25 10:38Z: the published corpus (`github-content-sync`) carries no release notes. They used to sit in the engine's `resources/content/release-notes`, where the Brain's github services got them. neomjs/neo#19157 moves the engine's authored notes to `.github/RELEASE_NOTES/`, and its decision says the corpus carries the notes too (neomjs/neo#19157 issuecomment-5831178267, item 3). Its window order puts this leaf first (issuecomment-5832172298).

## The Problem

- `ai/scripts/maintenance/syncGithubWorkflow.mjs` `--corpus-only` "emits the three conversation facets" (issues, pulls, discussions) plus the release reference used for bucketing. The `releaseNotes` facet (`ReleaseNotesSyncer.syncNotes`, `ai/services/github-workflow/SyncService.mjs:238`) runs only in the full mode that writes into the engine.
- So a reader that sees only the corpus has no notes. `issueSync.releaseNotesDir` resolves `<originRoot>/release-notes` (`ai/mcp/server/github-workflow/configBase.mjs:352`), which is a folder the corpus does not carry. The engine's local portal, which reads content from a declared root (neomjs/neo#19166), has none either.
- **Running that facet in corpus mode is not enough on its own** (found at intake, 2026-09-25):
  - The published corpus metadata caches 1196 releases as `{publishedAt}` only. None has a `contentHash` or a body (`neo/.sync-metadata.json`).
  - `ReleaseNotesSyncer.fetchAndCacheReleases` takes its fast path whenever the cached latest release matches GitHub's, and then hands `syncNotes` body-less `metadataOnly` releases.
  - `syncNotes` skips each of those with a warning, since no `contentHash` is cached, but still writes a `release-notes/_index.json` that lists every in-window release.
  - The first corpus run would therefore publish an index of about 168 notes, no note files, and exit 0. A test against an empty scratch root cannot see this, because an empty cache never takes the fast path.
- `syncNotes` also stores a release's new `contentHash` before it writes the file, and it only logs a failed write. So a note that failed once is skipped as unchanged on every later run.
- **Through `SyncService`, it has written nothing since 2026-07-26** (found in the red run). neomjs/neo `322064b1eb` (#16011) has the `releases` facet set `metadata.releases = ReleaseNotesSyncer.releases`, the same objects. The early hash assignment then makes each release its own cache entry, so the "unchanged" check always passes. A corpus run with only the flag flipped writes `release-notes/_index.json`, no notes, and exits 0. No neo release has been published since `13.1.0` (07-03), so no note is lost yet. The 13.2 post-release sync would have been the first miss.

## The Architectural Reality

- `ReleaseNotesSyncer` fetches every GitHub Release over GraphQL and writes chunked markdown through `contentBucketDir` / `chunkNumberFor`, the same layout as the engine's `resources/content/release-notes/chunk-N/`.
- `MetadataManager` persists each release as `{publishedAt, contentHash}`. The engine's full mode caches hashes, so its fast path is sound. The corpus never ran `syncNotes`, so it has none.
- The engine creates each Release from its authored note (neomjs/neo `buildScripts/release/publish.mjs:254`, `--notes-file`), so the synced copy is the published form of the note.
- The corpus publisher (neomjs/github-content-sync `.github/workflows/publish-corpus.yml`) takes the emitter's exit code as its whole verdict ("all four outcomes advanced") and publishes `<repo>/…` in one revision.

## The Fix

- `--corpus-only` also runs the `releaseNotes` facet, writing into `<root>/<repo>/release-notes/` with the same chunking as today, for every origin that has releases. `assertCorpusDestination` holds `releaseNotesDir` inside the origin root, as it does for the other facets.
- When notes are wanted, `fetchAndCacheReleases` takes its fast path only if every in-window release already has its note on disk and a cached `contentHash`. Otherwise it fetches the history with bodies.
- `syncNotes` reads the cached hash before anything assigns it, and records a `contentHash` only after the note's file is written. A release it cannot write, or one with no body, fails the facet instead of logging a warning, so the emitter exits nonzero and nothing is published.
- The Knowledge Base side, reading those notes from the corpus, is #490. It is a separate service boundary, and the root manifest cannot carry the notes: `ConversationCorpusSource` refuses a row it doesn't know.

## Acceptance Criteria

- [ ] A `--corpus-only` run over a scratch root whose metadata caches releases the way the published corpus does (`{publishedAt}` only, latest release current) writes `neo/release-notes/chunk-N/v*.md` for every in-window release. An origin without releases gets none.
- [ ] A release whose note cannot be written, or which has no body, makes the run exit nonzero, and that release's `contentHash` is not cached.
- [ ] With `originRoot` on the corpus's `neo/`, `issueSync.releaseNotesDir` names an existing folder. The corpus guard holds that folder inside the origin, as it does for the other facets.
- [ ] Post-merge, flagged: the next published corpus revision carries `neo/release-notes/`.

## Out of Scope

- Moving the engine's authored notes (neomjs/neo#19157).
- The full-mode materialization of notes into the engine, which retires with neomjs/neo#19159.
- The publisher's header comment in `github-content-sync`, which says "four outcomes". That comment changes in its own repository.

## Related

neomjs/neo#19157 · neomjs/neo#19166 · #459 · neomjs/github-content-sync `publish-corpus.yml`

Live latest-open sweep: the latest 20 open `neomjs/neo-agent-brain` issues at 2026-09-25T12:13:50Z, plus `release notes corpus` across Brain issues (all states) and the `github-content-sync` issue list. No equivalent; #459 is the related fallback ticket.
A2A in-flight sweep: no claim on this scope. I offered it to @neo-opus-vega, the #17416 steward, at 10:50Z, and got no claim.
MC sweep: `query_raw_memories("published corpus carries no release notes, corpus-only emits three conversation facets, ReleaseNotesSyncer release-notes github-content-sync")`, 5 results, no prior decision.
Own-assignment sweep: none on this surface.

Decision Record impact: none

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201

Authored by 🖖 **Grace** · `@neo-opus-grace` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-09-25T12:14:36Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T12:14:37Z @neo-opus-grace added the `enhancement` label
- 2026-09-25T12:14:37Z @neo-opus-grace added the `ai` label
- 2026-09-25T12:14:37Z @neo-opus-grace added the `agent-os` label
- 2026-09-25T14:13:56Z @neo-opus-grace cross-referenced by #490
- 2026-09-25T14:14:02Z @neo-opus-grace marked this issue as blocking #490
- 2026-09-25T14:19:11Z @neo-opus-grace cross-referenced by PR #491
- 2026-09-25T14:40:44Z @neo-opus-grace referenced in commit `c55289a` - "fix(github-workflow): an origin with no release in window gets no release-notes folder (#485)

syncNotes created <origin>/release-notes/ and wrote an empty _index.json even with nothing in window, which the publisher's release-less origins hit on their first run. It now returns before the mkdir."
- 2026-09-25T14:54:58Z @neo-opus-grace referenced in commit `103df18` - "feat(github-workflow): the release-notes index names each note's path (#485)

Each item now carries the note's content-root-relative path, the form the corpus root index uses. A reader such as #490 takes a note's identity from the index instead of re-deriving the producer's filename prefix: neo's tags are bare (13.1.0), while its files are v13.1.0.md."
- 2026-09-25T15:05:16Z @neo-opus-grace cross-referenced by PR #494

