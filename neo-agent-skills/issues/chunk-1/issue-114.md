---
id: 114
title: 'Consumers call the shared baseline by commit SHA, not by a published release — tag every publish and call it by version'
state: OPEN
labels:
  - enhancement
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T19:33:55Z'
updatedAt: '2026-09-24T20:09:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/114'
author: neo-opus-grace
commentsCount: 1
parentIssue: null
subIssues:
  - '[ ] 116 Every publish is tagged, and the shared baseline refuses a caller that is not at a release tag'
subIssuesCompleted: 0
subIssuesTotal: 1
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Consumers call the shared baseline by commit SHA, not by a published release — tag every publish and call it by version

## Context

Operator, 2026-09-24: *"we want npm versions. published skill releases. NEVER EVER SHA values. if there are SHA values, that must get changed."*

Two consumers call the shared baseline by commit SHA:
- neomjs/neo `.github/workflows/pr-baseline.yml:73`: `reusable-pr-baseline.yml@793b580587de…`;
- neomjs/neo-agent-institution `.github/workflows/shared-pr-baseline.yml:33`: `reusable-pr-baseline.yml@6b009521fac6…`.

This repository has **0 tags and 0 GitHub releases**, so nothing but a SHA or a branch can name a release today. Dependabot therefore cannot move these pins. That is why the neo pin-move PRs (neomjs/neo#19119, #19142) were written by hand, while the npm dependency already moves by dependabot.

## The Problem

A consumer's baseline is named by an opaque commit, not by the published release it is meant to run. The SHA carries no version, cannot be told apart from an unreleased commit, and has to be moved by hand on every publish.

## The Fix

1. **Tag every publish.** A `postpublish` script tags the published commit `v${npm_package_version}` and pushes the tag, so `npm publish` and the release tag cannot diverge. The first tag is `v0.1.17`, the publish that carries #111 and #113.
2. **Call by version.** Each consumer calls `neomjs/neo-agent-skills/.github/workflows/reusable-pr-baseline.yml@vX.Y.Z`. At that tag, the workflow's `SKILLS_VERSION` pins install `neo-agent-skills@X.Y.Z` (asserted equal by `test-reusable-pr-baseline`), so the tag and the npm release are the same version.
3. **Refuse a SHA or a branch.** A first job in the reusable baseline fails unless `job.workflow_ref` ends in `@refs/tags/v<semver>`, so a SHA or branch caller reds at once. The workflow header comment's "immutable `uses:` coordinate" becomes "a published release tag".
4. **Dependabot moves the tag.** Dependabot's `github-actions` ecosystem is already enabled daily in both consumers. It moves the `@vX.Y.Z` reference on each release, beside the npm bump: no hand-written pin PRs.

## Acceptance Criteria

- [ ] **AC-1:** Publishing creates and pushes `v<version>` on the published commit. `v0.1.17` exists on the commit npm 0.1.17 was built from.
- [ ] **AC-2:** The reusable baseline fails when called at a SHA or a branch and passes at a `v<semver>` tag. The contract test covers both.
- [ ] **AC-3:** No workflow in any neomjs consumer references `neo-agent-skills` by SHA: neo and the Institution call `@v0.1.17`.
- [ ] **AC-4:** The first release after `v0.1.17` receives a dependabot PR moving the `@vX.Y.Z` reference. If it does not, the ticket reopens with the reason.

## Out of Scope

- The semver policy (what counts as patch, minor or major) and 1.0.0: a separate Discussion.
- `v*` tag protection: a repository ruleset setting, which is the operator's to apply.

## Supersedes

#38 proposed installing the guards from `job.workflow_sha`, which moves toward SHAs; it is closed as not planned.

## Related

#56 (merged substrate is not coupled to a published version) · #38 · #27 · neomjs/neo#19119 · neomjs/neo#19142

Sweeps at 19:33Z:
- **Latest open:** of the latest 20 open issues, three touch versions or pins: #38 (superseded here), #56 (publish coupling, which this narrows for tags) and #90 (a commit-time guard installer, a different surface).
- **Consumer code search:** `neo-agent-skills/.github/workflows` finds the two callers above and nothing else.
- **A2A:** no claim.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T19:33:56Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T19:33:57Z @neo-opus-grace added the `enhancement` label
- 2026-09-24T19:33:57Z @neo-opus-grace added the `build` label
- 2026-09-24T19:33:58Z @neo-opus-grace cross-referenced by #38
- 2026-09-24T19:39:14Z @neo-opus-grace cross-referenced by PR #115
### @neo-opus-grace - 2026-09-24T19:39:47Z

## Contract Ledger (T3), for PR #115, which resolves the leaf #116. #114 stays open for row 3.

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `npm publish` (`postpublish` → `scripts/tag-release.mjs`) | the operator, 2026-09-24: "npm versions … NEVER EVER SHA values" | Puts `v<version>` (annotated) on origin at the published commit. It is idempotent: a tag already on origin at HEAD is a no-op. | A tag at another commit, locally or on origin, is refused. A dirty tree is refused before tagging. A failed push exits 1 and names the rerun that finishes the release, since npm cannot republish. | README | `test-tag-release.mjs` against a bare origin: tag and push, idempotent rerun, failed push, recovery rerun, dirty tree, stale tag. Mutations on the dirty guard and on the local-tag handling red it. |
| `reusable-pr-baseline.yml` job `Release ref` | same | Passes only when `job.workflow_ref` ends in `@refs/tags/vX.Y.Z`. | A SHA or branch caller fails with a message naming the ref it was called at. | Workflow header, README | The contract test requires the job, `job.workflow_ref` and the pattern, and 3 mutations red it. The bash check passes `v0.1.17` and refuses a SHA and `refs/heads/dev`. |
| Consumer callers (neo `pr-baseline.yml:73`, Institution `shared-pr-baseline.yml:33`) | same | Call `…/reusable-pr-baseline.yml@vX.Y.Z`; dependabot's `github-actions` ecosystem moves the tag on each release. | Until they switch, callers still run their pinned SHA's old workflow, which has no `Release ref` job, so nothing reds before they move. | — | Residual, after the first tag exists: #114 AC-3 and AC-4. |

🖖 Grace (Claude Opus 5.5, Claude Code) · session 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2


- 2026-09-24T20:07:50Z @neo-opus-grace cross-referenced by #116
- 2026-09-24T20:07:52Z @neo-opus-grace added sub-issue #116
- 2026-09-24T20:16:26Z @neo-opus-grace referenced in commit `a8b43ba` - "feat(release): every publish is tagged, and the shared baseline runs only at a release tag (#114)

postpublish tags the published commit v<version> and pushes it, refusing a dirty tree or an existing tag. The reusable baseline's first job, Release ref, fails unless job.workflow_ref ends in a vX.Y.Z tag, so a SHA or branch caller reds. The contract test gains the job and three mutations; test-tag-release covers the script and joins npm test and CI."
- 2026-09-24T20:18:24Z @neo-opus-grace cross-referenced by #19197
- 2026-09-24T20:19:06Z @neo-opus-grace cross-referenced by PR #19198
- 2026-09-24T20:19:46Z @neo-opus-grace cross-referenced by #189
- 2026-09-24T20:20:30Z @neo-opus-grace cross-referenced by PR #190
- 2026-09-24T20:30:26Z @neo-opus-grace cross-referenced by #39

