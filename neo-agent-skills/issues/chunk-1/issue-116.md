---
id: 116
title: 'Every publish is tagged, and the shared baseline refuses a caller that is not at a release tag'
state: OPEN
labels:
  - enhancement
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T20:07:48Z'
updatedAt: '2026-09-24T20:16:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/116'
author: neo-opus-grace
commentsCount: 1
parentIssue: 114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Every publish is tagged, and the shared baseline refuses a caller that is not at a release tag

## Context

This is the package side of #114, split out so a PR can close it while #114 stays open for what happens after the first tag: the consumer switch and dependabot's move (#114 AC-3 and AC-4). The operator's rule, 2026-09-24: *"we want npm versions. published skill releases. NEVER EVER SHA values."*

## The Fix

1. **Tag every publish.** `postpublish` runs `scripts/tag-release.mjs`, which puts `v<version>` on origin at the published commit. It runs after the registry already holds the version, so every path is loud and safe to rerun:
   - a tag already on origin at HEAD is a no-op;
   - a tag marking another commit is refused;
   - a dirty tree is refused before tagging;
   - a failed push exits 1, and names the rerun that finishes the release.
2. **Refuse anything but a release tag.** The reusable baseline's first job, `Release ref`, fails unless `job.workflow_ref` ends in `@refs/tags/vX.Y.Z`.

## Acceptance Criteria

- [ ] **AC-1:** Against a real origin, the script tags and pushes a clean published commit and is a no-op on rerun. It fails loudly on a failed push, and a later rerun finishes it. It refuses a dirty tree, and a tag that marks another commit.
- [ ] **AC-2:** The reusable baseline fails when called at a SHA or a branch and passes at a `vX.Y.Z` tag. The contract test covers the job, its context and its pattern.

## Parent

#114 (owns the consumer switch and the dependabot read).

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T20:07:49Z @neo-opus-grace added the `enhancement` label
- 2026-09-24T20:07:49Z @neo-opus-grace added the `build` label
- 2026-09-24T20:07:50Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T20:07:52Z @neo-opus-grace added parent issue #114
- 2026-09-24T20:08:17Z @neo-opus-grace referenced in commit `398e9af` - "fix(release): a failed tag push is loud and a rerun finishes the release (#116)

postpublish runs after the registry holds the version, so the script is idempotent: a tag already on origin at HEAD is a no-op, a tag marking another commit is refused, a dirty tree is refused before tagging, and a failed push exits 1 naming the rerun. The test runs against a real bare origin: tag and push, idempotent rerun, failed push, recovery, dirty tree, stale tag."
- 2026-09-24T20:08:42Z @neo-opus-grace cross-referenced by PR #115
- 2026-09-24T20:09:17Z @neo-opus-grace cross-referenced by #114
- 2026-09-24T20:16:26Z @neo-opus-grace referenced in commit `36ea613` - "fix(release): a failed tag push is loud and a rerun finishes the release (#116)

postpublish runs after the registry holds the version, so the script is idempotent: a tag already on origin at HEAD is a no-op, a tag marking another commit is refused, a dirty tree is refused before tagging, and a failed push exits 1 naming the rerun. The test runs against a real bare origin: tag and push, idempotent rerun, failed push, recovery, dirty tree, stale tag."
- 2026-09-24T20:16:26Z @neo-opus-grace referenced in commit `2c8e9f1` - "fix(release): a dry run tags nothing, and a caller must be at this release's own tag (#116)

npm publish --dry-run runs postpublish with npm_config_dry_run=true (measured on npm 11.12.1), so tag-release exits before touching git. Release ref now requires @refs/tags/v${SKILLS_VERSION}, its own pin, which the drift assertion holds to package.json: v999.999.999 or v01.2.3 no longer pass. The pin mutation that assumed the archaeology pin came first is scoped to its job."
- 2026-09-24T20:16:26Z @neo-opus-grace referenced in commit `792608b` - "chore(release): the release tags ship as 0.1.18, since 0.1.17 is published (#116)

0.1.17 (#111, #113) is on npm; package.json, the lockfile and all seven SKILLS_VERSION pins, the Release ref pin included, move to 0.1.18."
### @neo-opus-grace - 2026-09-24T20:16:58Z

## Contract Ledger (T3), for PR #115, which resolves this issue

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `npm publish` → `postpublish` → `scripts/tag-release.mjs` | the operator, 2026-09-24: "npm versions … NEVER EVER SHA values"; #114 | Puts an annotated `v<version>` on origin at the published commit. It is idempotent: a tag already on origin at HEAD is a no-op. | `npm publish --dry-run` (which still runs postpublish, with `npm_config_dry_run=true`, measured on npm 11.12.1) exits 0 and tags nothing. A tag at another commit, locally or on origin, is refused. A dirty tree is refused before tagging. A failed push exits 1 and names the rerun, since npm cannot republish. | README | `test-tag-release.mjs`, 7 cases against a bare origin: dry run, tag and push, idempotent rerun, failed push, recovery rerun, dirty tree, stale tag. Mutations on the dirty guard and on the local-tag handling red it. |
| `reusable-pr-baseline.yml` job `Release ref` | same | Passes only when `job.workflow_ref` ends in `@refs/tags/v${SKILLS_VERSION}`, its own pin. That is the npm version every job installs, so tag and package are one number. | Any other ref fails and names the ref it was called at: a SHA, a branch, another version, or a non-canonical tag such as `v01.2.3`. | Workflow header, README | The contract test requires the job, `job.workflow_ref` and the equality check, and the drift assertion holds the pin to `package.json`. Mutations: job removed, the caller's ref read instead, the check loosened to any semver tag, the pin drifted. The bash check passes only `v0.1.17` against pin 0.1.17. |
| Release | the per-publish version rule | This ships as 0.1.18: `package.json`, the lockfile and all seven `SKILLS_VERSION` pins. 0.1.17 is published, with `gitHead` `f7b29e7`, and tagged `v0.1.17` by hand because it predates this script. | — | — | `test-reusable-pr-baseline` is green with 7 pins. `git ls-remote origin refs/tags/v0.1.17` peels to `f7b29e7`. |

🖖 Grace (Claude Opus 5.5, Claude Code) · session 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2



