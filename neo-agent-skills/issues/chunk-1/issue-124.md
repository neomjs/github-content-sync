---
id: 124
title: The release-notes workflow still names the retired resources/content path
state: CLOSED
labels:
  - documentation
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-29T09:50:36Z'
updatedAt: '2026-09-29T13:53:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/124'
author: neo-opus-vega
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
closedAt: '2026-09-29T13:53:54Z'
---
# The release-notes workflow still names the retired resources/content path

## Context

neomjs/neo#19159 deletes the engine's `resources/content`. Release notes are authored flat in `.github/RELEASE_NOTES/`, and `publish.mjs` keeps each one as the archive. neomjs/neo-agent-brain#623 stops the post-release lifecycle from syncing and committing into the engine.

## The Problem

Three skill references still point authors at the old tree:
- `release-notes/references/release-notes-workflow.md` §6. It names `resources/content/release-notes/v{version}.md` as the authoring surface. It describes publish.mjs deleting the note and the post-release sync re-materializing it under `chunk-N/`. It describes the orphan guard policing a flat root beside `chunk-N`, and the husky sync-guard classifying the notes as sync data. None of that holds after the deletion.
- `update-roadmap/references/update-roadmap-workflow.md:36` routes shipped history to `resources/content/release-notes/`.
- `guide-authoring/references/guide-authoring-bar.md:17` cites `resources/content/release-notes/chunk-2/v13.0.0.md`.

## The Architectural Reality

- The §1 scope recipe symlinks a corpus checkout into a temp `resources/content/` for `analyzeClosedSinceRelease.mjs`. That script still reads that layout under its own root, so the recipe stays.
- The notes' remaining guards are `PublishReleaseNoteOrphan.spec.mjs` (authored path, kept after the release, flat, one file per version) and the release-note link scan in `Generate.spec.mjs`.

## The Fix

Rewrite §6's items 1, 4, 5 and 6 to the new lifecycle, and repoint the two citations.

## Acceptance Criteria

- [ ] **AC-1** No skill reference names `resources/content/release-notes` as an authoring surface or a citation target.
- [ ] **AC-2** §6 describes the lifecycle `publish.mjs` runs after neomjs/neo#19159: requires, appends the atomic hash, creates the release, keeps the file.

## Out of Scope

- The §1 scope recipe (still correct, see above).

## Related

neomjs/neo#19159 · neomjs/neo#19157 · neomjs/neo-agent-brain#623

Live latest-open sweep: latest 20 open issues here at 2026-09-29T09:49Z, none equivalent. A2A: no competing claim.

Origin Session ID: db0e34f7-9d0f-4799-a2c2-3a5033f8bc9a

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-29T09:50:37Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-29T09:50:38Z @neo-opus-vega added the `documentation` label
- 2026-09-29T09:50:38Z @neo-opus-vega added the `ai` label
- 2026-09-29T09:50:57Z @neo-opus-vega cross-referenced by #19159
- 2026-09-29T09:51:10Z @neo-opus-vega cross-referenced by #19157
- 2026-09-29T09:56:26Z @neo-opus-vega cross-referenced by PR #624
- 2026-09-29T10:01:15Z @neo-opus-vega cross-referenced by PR #125
- 2026-09-29T10:02:04Z @neo-opus-vega referenced in commit `e00d6a0` - "chore(release): bump to 0.1.22 (#124)"
- 2026-09-29T10:12:07Z @neo-preview cross-referenced by PR #19322
- 2026-09-29T13:53:54Z @tobiu referenced in commit `0c0209e` - "Merge pull request #125 from neomjs/vega/124-release-notes-path

docs(release-notes): the notes are authored and kept in the engine's .github/RELEASE_NOTES (#124)"
- 2026-09-29T13:53:54Z @tobiu closed this issue

