---
id: 56
title: 'Every PR bumps the version, every merge publishes it, and nothing else writes it'
state: CLOSED
labels:
  - enhancement
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-07T00:14:46Z'
updatedAt: '2026-09-25T16:46:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/56'
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
closedAt: '2026-09-25T16:46:11Z'
---
# Every PR bumps the version, every merge publishes it, and nothing else writes it

> **Amended 2026-09-25 by the author, at the operator's direction on PR #119.** Paraphrased: a version is written by hand exactly once, in the PR's `package.json`. Each PR carries a new version, CI publishes it to npm on merge as the engine does, Dependabot brings it to the consumers, and everything else reads it from `package.json`. So:
> - the bump arm covers every PR;
> - the npm credential blocker is gone, since the engine publishes through npm trusted publishing (OIDC), with no secret;
> - the seven `SKILLS_VERSION` literals are in scope (#38's subject);
> - #114's moving `@v0` tag is withdrawn, and PR #119 is dropped.
>
> The original body is in the edit history.

## Context

Nothing in this repository couples a merged change to a published version. Release is a manual `npm publish` from outside the repository, and `postpublish` (`scripts/tag-release.mjs`) then tags it. That missing coupling has already produced two tickets, from opposite directions:
- #27: the baseline pinned a `SKILLS_VERSION` npm did not have;
- #44: twelve merged commits sat under an unbumped `0.1.3`.

## The Problem

1. **No publish trigger.** A merged bump does nothing until someone publishes by hand.
2. **No bump enforcement.** A PR can merge without a version change, and nothing objects.
3. **The version is written by hand in eight places.**
   - They are `package.json` plus seven `SKILLS_VERSION: '<x.y.z>'` literals in `reusable-pr-baseline.yml`.
   - `scripts/test-reusable-pr-baseline.mjs` asserts they are equal, so every release edits the workflow by hand.

## The Fix

- **Every PR bumps.** `scripts/check-version-bump.mjs` fails a PR in these two cases. It runs in `skill-corpus.yml`.
  - Its `package.json` version is not greater than its base's.
  - npm already has that version.
- **Every merge publishes.** `.github/workflows/publish.yml` runs on each push to `dev`.
  - It tests, then publishes the `package.json` version through npm trusted publishing (`id-token: write`, as the engine's `npm-publish.yml` does).
  - `postpublish` (`scripts/tag-release.mjs`, unchanged) pushes `v<version>` at that commit.
  - A merge whose version npm already has fails the job and names the commit npm does not carry.
- **The version is read, never written.**
  - A `Skills version` job in `reusable-pr-baseline.yml` reads `package.json` at `job.workflow_sha` and outputs it.
  - `Release ref` compares the caller's tag with that output, and every install uses it.
  - The contract test fails on any version literal in the workflow.
- **Consumers stay as they are.** Dependabot already opens a `neo-agent-skills` npm bump in each consumer on its daily run, and it bumps tag-shaped `uses:` refs (#80).
  - GitHub has no API that starts a Dependabot run, so a publish cannot trigger one directly.
  - The next scheduled run proposes the new version.

## Operator action (credentials)

On npmjs.com, add a trusted publisher to `neo-agent-skills`:
- provider: GitHub Actions;
- repository: `neomjs/neo-agent-skills`;
- workflow: `publish.yml`.

Until that exists, the publish job fails on merge and names the missing publisher.

## Acceptance Criteria

- [ ] **AC-1:** A PR whose `package.json` version is not greater than its base's fails CI, and the failure names both versions. So does a PR whose version npm already has. A PR with a new, greater version passes.
- [ ] **AC-2:** A merge to `dev` publishes its `package.json` version and pushes `v<version>` at that commit. A merge whose version npm already has fails and names the commit.
- [ ] **AC-3:** `reusable-pr-baseline.yml` holds no version literal, and the contract test fails when one is added. Red control: seed a literal, show the test red, then restore it.
- [ ] **AC-4 (post-merge):** after the first CI publish, `npm view neo-agent-skills version` equals `dev`'s `package.json` version, and origin has the tag.

## Out of Scope

- Consumer `dependabot.yml` changes (a skills-specific schedule, #80's github-actions exclusion).
- The engine's release process.

Decision Record impact: none.

Related: #38 (the literals; closed, absorbed here) · #80 · #114 (superseded) · #119 (dropped) · #27 · #44 · #14

Origin Session ID: d2d30528-b6fe-423b-86ce-ab945396a201

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-07T00:14:48Z @neo-opus-grace added the `enhancement` label
- 2026-09-07T00:14:48Z @neo-opus-grace added the `agent-os` label
- 2026-09-07T00:14:58Z @neo-opus-grace cross-referenced by #44
- 2026-09-07T00:16:24Z @neo-opus-grace cross-referenced by PR #55
### @neo-opus-grace - 2026-09-07T00:16:45Z

**Correction: this is a prerequisite of PR #55, not a follow-up to it.**

The repo's own guard settled it. `test-reusable-pr-baseline.mjs` asserts `SKILLS_VERSION` equals `package.json`'s version at all three install sites, so a bump cannot merge without moving the three pins in `reusable-pr-baseline.yml` with it. That means merging a bump puts `neo-agent-skills@<unpublished>` into three `npm install` lines that every consumer repository calls — **verbatim #27**, already closed COMPLETED once.

So merge-now-publish-later is not an available ordering. Either the publish happens from the branch before the merge, or this ticket lands first and the two stop being separable. That moves this from "nice to have, prevents future drift" to "the thing that makes a version bump safe at all".

🖖 @neo-opus-grace


- 2026-09-08T07:51:41Z @neo-opus-grace cross-referenced by #18465
- 2026-09-08T07:53:11Z @neo-opus-grace cross-referenced by PR #18466
- 2026-09-08T15:25:19Z @neo-opus-grace cross-referenced by PR #18485
- 2026-09-16T08:58:36Z @neo-opus-vega cross-referenced by #80
- 2026-09-16T09:21:02Z @neo-opus-vega cross-referenced by #81
- 2026-09-16T10:32:28Z @neo-opus-ada cross-referenced by PR #82
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90
- 2026-09-24T19:33:56Z @neo-opus-grace cross-referenced by #114
- 2026-09-24T20:14:42Z @neo-gpt cross-referenced by PR #115
- 2026-09-25T15:40:28Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-25T15:40:29Z @neo-opus-grace changed title from **Nothing couples merged substrate to a published version, and this is the second ticket it has produced** to **Every PR bumps the version, every merge publishes it, and nothing else writes it**
- 2026-09-25T15:40:43Z @neo-opus-grace cross-referenced by PR #119
- 2026-09-25T15:40:48Z @neo-opus-grace cross-referenced by #38
- 2026-09-25T15:51:17Z @neo-opus-grace cross-referenced by PR #120
- 2026-09-25T16:46:11Z @tobiu referenced in commit `d7e559d` - "Merge pull request #120 from neomjs/grace/56-publish-on-merge

feat(release): every PR bumps the version, every merge publishes it, and nothing else writes it (#56)"
- 2026-09-25T16:46:11Z @tobiu closed this issue

