---
id: 623
title: The post-release lifecycle re-creates the engine's retired content mirror
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-09-29T09:50:25Z'
updatedAt: '2026-09-29T11:23:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/623'
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
closedAt: '2026-09-29T11:23:33Z'
---
# The post-release lifecycle re-creates the engine's retired content mirror

## Context

neomjs/neo#19159 deletes the engine's `resources/content` (operator, 2026-09-29). The corpus publishes from `neomjs/github-content-sync`, and the engine's release notes move to `.github/RELEASE_NOTES/`. `publish.mjs` keeps each note there as the archive and no longer deletes it after the release.

## The Problem

`ai/scripts/lifecycle/postReleaseSync.mjs` is the second release runbook command, and it still writes the mirror. After the Knowledge Base upload it runs `GH_SyncService.runFullSync()` with the engine checkout as cwd, regenerates the ticket index, then commits with `git add .` and pushes to the engine's `dev`. After the deletion, the first post-release run would recreate the tree and push it back.

Its preflight is also out of step: the only dirty path it admits is the deletion of `resources/content/release-notes/v<version>.md`, which `publish.mjs` no longer performs.

## The Architectural Reality

- `postReleaseSync.mjs`, main(): stage 2 (sync and ticket index) and stage 3 (collision assertion, broad commit, push). Its import of `findLogicalIdentityCollisions` from the engine package serves only that commit.
- `postReleasePreflight.mjs`: `assertAdmissibleStartingState` admits a clean tree or the note's deletion. `assertOnDevBranch` justifies itself by the archive commit.
- The Knowledge Base upload (`ai/scripts/maintenance/uploadKnowledgeBase.mjs`) packages the KB collection into a release artifact and reads nothing under `resources/content`.
- Structure map: N/A. No file is added or moved.

## The Fix

- `postReleaseSync.mjs` keeps the preflight and the Knowledge Base upload. Stages 2 and 3, the collision assertion and its import are removed.
- The preflight admits only a clean tree. `assertOnDevBranch` stays, because the upload must read the released `dev` state; its message says so.
- Specs: the note-deletion arms become refusals, and `postReleaseSync.spec.mjs` pins that the lifecycle neither syncs nor commits.

Decision Record impact: none.

## Acceptance Criteria

- [ ] **AC-1** `postReleaseSync.mjs` holds no `runFullSync`, no `git add` / `commit` / `push`, and no engine logical-identity import. The Knowledge Base upload still runs after the preflight.
- [ ] **AC-2** `assertAdmissibleStartingState` passes only a clean tree and refuses a staging-note deletion by name.
- [ ] **AC-3** The lifecycle unit specs pass.

## Out of Scope

- The Knowledge Base concept route over the engine pin (#474).
- Renaming the script or its `ai:post-release-sync` alias.

## Related

neomjs/neo#19159 · neomjs/neo#19157 · #474

Live latest-open sweep: latest 20 open issues here at 2026-09-29T09:49Z, none equivalent; org search (`post-release`, `RELEASE_NOTES`): only neomjs/neo#19157 and #19159. A2A: no competing claim. MC: the operator's deletion decision (2026-09-23, 2026-09-29), with no conflicting decision.

Origin Session ID: db0e34f7-9d0f-4799-a2c2-3a5033f8bc9a

Authored by Vega (Claude Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-09-29T09:50:26Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-29T09:50:26Z @neo-opus-vega added the `enhancement` label
- 2026-09-29T09:50:26Z @neo-opus-vega added the `ai` label
- 2026-09-29T09:50:26Z @neo-opus-vega added the `build` label
- 2026-09-29T09:50:37Z @neo-opus-vega cross-referenced by #124
- 2026-09-29T09:50:57Z @neo-opus-vega cross-referenced by #19159
- 2026-09-29T09:51:10Z @neo-opus-vega cross-referenced by #19157
- 2026-09-29T09:52:39Z @neo-opus-vega cross-referenced by PR #19322
- 2026-09-29T09:56:26Z @neo-opus-vega cross-referenced by PR #624
- 2026-09-29T10:12:56Z @neo-preview cross-referenced by PR #12
- 2026-09-29T10:13:25Z @neo-preview cross-referenced by PR #125
- 2026-09-29T11:23:33Z @tobiu referenced in commit `e00e6b1` - "Merge pull request #624 from neomjs/vega/623-post-release-no-engine-writes

fix(lifecycle): the post-release step writes nothing into the engine checkout (#623)"
- 2026-09-29T11:23:34Z @tobiu closed this issue

