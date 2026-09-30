---
id: 129
title: 'The CI publish has failed on every merge since 09-25, so npm is still at 0.1.19'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T18:32:42Z'
updatedAt: '2026-09-30T18:40:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/129'
author: neo-opus-grace
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
closedAt: '2026-09-30T18:40:53Z'
---
# The CI publish has failed on every merge since 09-25, so npm is still at 0.1.19

## Context

#56 (mine) made every merge to `dev` publish through npm trusted publishing. Its post-merge AC-4 (after the first CI publish, npm shows `dev`'s version and origin has the tag) was never verified, and the publish has failed on every run:

| run | when | error |
|---|---|---|
| 36162859030 | 2026-09-25 | `E404 Not Found - PUT …/neo-agent-skills` |
| 36192136459 | 2026-09-25 | `E404` |
| 36578539485 | 2026-09-29 | `E404` |
| 36758610375 | 2026-09-30, dispatched after @tobiu configured the trusted publisher on npmjs.com | `E403 … OIDC permission denied for this action` |

npm's `latest` is 0.1.19 and the tags stop at `v0.1.19`, so 0.1.20–0.1.22 never shipped.

**Sweep attestations (2026-09-30):** latest 20 open issues here: no publish ticket; search for publish/OIDC/E403/E404: none. Memory Core: no prior decision on `publishConfig`.

## The Problem

The engine's `npm-publish.yml` publishes `neo.mjs` through the same mechanism: `setup-node@v7`, Node 24, `registry-url`, `id-token: write`, no `NODE_AUTH_TOKEN`. Its last run (28684516218, 13.1.0) logged *"Publishing to https://registry.npmjs.org/ with tag latest and default access"*. Ours logs *"… with tag latest and **public** access"*, because `package.json` declares `"publishConfig": {"access": "public"}`; the engine declares none. Everything else in the two runs matches, provenance signing included.

For an unscoped package the access is always public, so the field changes nothing except the request, which now asks for an access setting the trusted-publishing token was denied.

## The Fix

Remove `publishConfig` from `package.json`. The merge's own publish run is the witness.

## Acceptance Criteria

- [ ] AC-1: `package.json` declares no `publishConfig`, and the packed file list is unchanged.
- [ ] AC-2 (post-merge): the merge's publish run passes: `npm view neo-agent-skills version` equals this version, and origin has its tag.

## Out of Scope

- Making a failed publish visible before five days pass: `check-version-bump.mjs` already queries npm and could also require the base's version to be published. A separate change.

## Related

- #56 — publish-on-merge, whose AC-4 this closes out.
- neomjs/neo `.github/workflows/npm-publish.yml` — the working reference.
- #100 / PR #127 — the next release, which Brain #644 needs on npm.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636


## Timeline

- 2026-09-30T18:32:44Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T18:32:44Z @neo-opus-grace added the `bug` label
- 2026-09-30T18:32:44Z @neo-opus-grace added the `ai` label
- 2026-09-30T18:34:27Z @neo-opus-grace cross-referenced by PR #130
### @neo-opus-grace - 2026-09-30T18:40:52Z

Resolved on npmjs.com, not in this repository.

**The cause:** the trusted publisher for `neo-agent-skills` allowed only a *staged* publish. The engine's entry allows staged and direct publish. @tobiu enabled direct publish, and the next run went through:

- run 36759447045 (2026-09-30 18:33Z): `Publish, then tag` green; npm answered "Your package is being processed and may take a few minutes to become available".
- the registry's `latest` became `0.1.22` at about 18:38Z, some four minutes after the publish returned.
- `tag-release: v0.1.22 at 0c0209e, pushed`.

**My hypothesis was wrong:** the successful run still published "with tag latest and public access", so `publishConfig` was never the cause. AC-1 is withdrawn and PR #130 is closed unmerged.

One observation, not a ticket: `postpublish` pushes the tag as soon as `npm publish` returns, a few minutes before npm serves the version. A consumer that resolves the new tag inside that window cannot install it yet. Dependabot's cadence makes the window harmless today.

🖖 Grace (Claude Opus 5.5, Claude Code)

- 2026-09-30T18:40:52Z @neo-opus-grace cross-referenced by #56
- 2026-09-30T18:40:53Z @neo-opus-grace closed this issue

