---
id: 256
title: 'Ship the CodeQL lane as a reusable workflow, extraction guard included'
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-ada
createdAt: '2026-11-01T18:26:55Z'
updatedAt: '2026-11-12T00:01:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/256'
author: neo-opus-ada
commentsCount: 2
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
closedAt: '2026-11-12T00:01:38Z'
---
# Ship the CodeQL lane as a reusable workflow, extraction guard included

## Context

On 2026-10-10 the operator enabled the `code_scanning` merge rule (tool `CodeQL`) in every repository's `dev` ruleset. Only the engine had a committed CodeQL workflow (`neomjs/neo/.github/workflows/codeql-analysis.yml`). Brain, devindex and Skills were moved to GitHub's Default Setup to satisfy the rule, and Institution was already on it. Default Setup never ran on the first external Brain pull request, neomjs/neo-agent-brain#1412. That PR sat `BLOCKED` with 20 green checks and no CodeQL run. The repair today is one copy of the engine's workflow per repository: neomjs/neo-agent-brain#1419 (for neomjs/neo-agent-brain#1418) and neomjs/neo-agent-institution#856 (for neomjs/neo-agent-institution#855).

## The Problem

1. **Drift.** Each copy is maintained by hand. Today's deadlock came from exactly that drift, and nobody noticed for three weeks because no fork PR arrived.
2. **No extraction guard outside the engine.** The engine's workflow pairs `Analyze` with an `extraction-guard` job (`buildScripts/util/check-codeql-extraction.mjs`, from neomjs/neo#15370, hardened in neomjs/neo#16075 and neomjs/neo#20035). The workflow comment states the reason: a file CodeQL cannot parse emits zero alerts and clears the alert gate, so "unparseable" and "clean" look the same. The Brain copy in neomjs/neo-agent-brain#1419 (head `017787b7`) has no guard. neomjs/neo-agent-brain#1418's Out of Scope routes a shared guard here, but no ticket existed. Default Setup had the same blind spot.
3. **Skills would deadlock a fork PR the same way.** On `dev`, Skills has no CodeQL workflow file. Its PRs carry Default-Setup `Analyze (actions)` / `Analyze (javascript-typescript)` checks (e.g. #254), and its ruleset requires `code_scanning`. Its last 100 PRs include no fork PR, so this is unobserved, but the configuration matches Brain's before neomjs/neo-agent-brain#1419. devindex is in the same state: no workflow file, and `code_scanning` is required.

## The Architectural Reality

- Skills already ships shared CI as reusable workflows: `.github/workflows/reusable-pr-baseline.yml` and `reusable-dependabot-automerge.yml`, which Brain calls at `@v0.1.68`. A CodeQL lane fits that precedent.
- Categories must stay `/language:<language>`. That is what Default Setup wrote and what neomjs/neo-agent-brain#1419 keeps, so each repository's alert history continues.
- Inside a called workflow, job names gain the caller's prefix. The guard reads the Analyze job's log by job, so the port has to resolve the called job's name.
- Advanced-Setup uploads are refused while Default Setup is on in the consuming repository. Switching it off is a repository setting the operator owns, and it is a merge-order gate for each adoption.

Design authority: n/a. This adds a shared lane and calls no existing behavior wrong. The guard's rationale is the engine record neomjs/neo#15370.

## The Fix

Add `.github/workflows/reusable-codeql.yml` with a `languages` input (JSON array). It runs `Analyze (<language>)` with `build-mode: none` and `category: "/language:<language>"`, with least-privilege permissions. It also runs the extraction-guard job ported from the engine. Skills then adopts the lane as its own CodeQL workflow, so the PR exercises it on itself.

## Acceptance Criteria

- [ ] `reusable-codeql.yml` exists with the `languages` input, per-language categories, least-privilege permissions, and the extraction-guard job.
- [ ] Skills calls it on `push` / `pull_request` to `dev` plus a weekly schedule. The PR's own run shows `Analyze` and the guard green, and the `CodeQL` check passes.
- [ ] A test proves the guard fails a run whose Analyze log reports an extraction error, matching the engine's neomjs/neo#15370 behavior.
- [ ] The PR states its merge-order gate: Default Setup must be off in the Skills settings (operator) before merge.

## Out of Scope

- Adopting the lane in Brain, Institution, devindex and the engine: one PR each, after a Skills release carries it.
- Changing any ruleset.

## Related

neomjs/neo-agent-brain#1412 · neomjs/neo-agent-brain#1418 / neomjs/neo-agent-brain#1419 · neomjs/neo-agent-institution#855 / neomjs/neo-agent-institution#856 · neomjs/neo#15370 · precedent `.github/workflows/reusable-pr-baseline.yml` · #338 (the first fork PR this unblocks; the operator chose this fix on 2026-11-11)

Live latest-open + closed sweep: checked latest 20 open + closed search (`codeql OR reusable workflow OR code scanning`; #154, #156 and #15 read as non-equivalent) at 2026-11-01T18:26Z. A2A claims (all read-states, last 12) were checked. The Memory Core rationale sweep covered the 2026-10-10 Default-Setup decision and today's #1412 diagnosis. No equivalent or prior exclusion found.

Owner: @neo-opus-ada (self-assigned 2026-11-11, after the operator chose this fix over a ruleset bypass for #338).

Decision Record impact: none.

Origin Session ID: 532727df-bad5-4b83-bf57-3878087ac8f0

Retrieval Hint: "CodeQL default setup fork PR deadlock reusable workflow extraction guard"

## Timeline

- 2026-11-01T18:26:57Z @neo-opus-ada added the `enhancement` label
- 2026-11-01T18:26:58Z @neo-opus-ada added the `ai` label
- 2026-11-01T18:26:58Z @neo-opus-ada added the `build` label
- 2026-11-01T18:27:07Z @github-actions added the `needs-design-authority` label
- 2026-11-01T18:29:16Z @neo-opus-ada cross-referenced by PR #1419
- 2026-11-01T18:45:41Z @neo-opus-ada cross-referenced by PR #856
- 2026-11-02T06:01:52Z @neo-opus-grace cross-referenced by #258
- 2026-11-11T15:20:32Z @neo-gemini-maryam cross-referenced by PR #338
- 2026-11-11T15:28:22Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-11T15:35:59Z @neo-opus-ada cross-referenced by PR #348
- 2026-11-11T15:36:23Z @neo-opus-ada removed the `needs-design-authority` label
- 2026-11-11T15:39:35Z @neo-opus-ada referenced in commit `19051b1` - "fix(ci): quote the CodeQL lane's languages description (#256)

An unquoted 'build-mode: none' inside the input description is a YAML mapping, so GitHub refused the workflow file."
### @neo-opus-ada - 2026-11-11T16:17:22Z

**Handover (session sunset; next session picks up today).** [PR #348](https://github.com/neomjs/neo-agent-skills/pull/348) is at `19051b1`. The lane runs, and both Analyze jobs finish their analysis. GitHub then refuses the upload while Default Setup is on ([status](https://github.com/neomjs/neo-agent-skills/pull/348#issuecomment-6800233832)). That is the merge-order gate, not a defect.

**Pickup:**
1. The operator switches CodeQL Default Setup off under Settings → Code security.
2. Re-run the failed jobs: `gh run rerun 43050566912 --failed --repo neomjs/neo-agent-skills`.
3. Once the checks are green, request Maryam as primary reviewer in GitHub and send her a 1:1. She asked to be pinged at green.
4. After the merge, #338 needs a rebase onto `dev` and a patch bump (`maintainer_can_modify: true`). Then the operator approves its workflow run once.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-11T18:35:14Z @neo-opus-ada referenced in commit `f82861f` - "chore(ci): merge dev 1.4.0 into the CodeQL lane and bump to 1.4.1 (#256)"
### @neo-opus-ada - 2026-11-11T18:42:19Z

**#348 pickup, updated 2026-11-11 18:4xZ.**

- **Head:** `f82861f`. Dev `b3a73a9` (#347) is merged in with a ticketed merge, and the version is 1.4.1, above dev's 1.4.0. `npm test` passes locally.
- **Still blocked on the operator:** turning CodeQL Default Setup off for this repository. Until then the lane's `CodeQL / Analyze (*)` jobs stay red, because uploads are refused.
- **After the switch:**
  1. Run `gh run rerun <latest CodeQL run> --failed --repo neomjs/neo-agent-skills`.
  2. At green, request review from Maryam on GitHub and send her a 1:1.
  3. After the merge, rebase #338 and give it a patch bump above 1.4.1.

**Owner:** Ada (next session).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

- 2026-11-11T21:10:19Z @neo-opus-ada referenced in commit `422a352` - "chore(ci): merge dev 1.4.3 into the CodeQL lane and bump to 1.4.4 (#256)"
- 2026-11-11T23:48:19Z @neo-opus-ada referenced in commit `92ca052` - "ci(codeql): skill-corpus runs the CodeQL suites (#256)

skill-corpus.yml runs each contract suite by name rather than through npm test, so the two suites this PR added never ran in CI. Both now have their own step, and the tarball check asserts that the guard and its two suites stay out of the consumer package."
- 2026-11-11T23:48:19Z @neo-opus-ada referenced in commit `c13dd64` - "fix(codeql): the guard matches only a job's last segment (#256)

ANALYZE_JOB matched an 'Analyze (' anywhere after a segment boundary, so a caller job named 'Analyze (CodeQL)' would have made the guard's own job read as an Analyze leg. The pattern now anchors to the last segment, and the spec pins that caller name."
- 2026-11-12T00:01:38Z @tobiu referenced in commit `c6e07e7` - "feat(ci): reusable CodeQL lane with extraction guard (#256) (#348)

* feat(ci): reusable CodeQL lane with extraction guard (#256)

Skills ships the CodeQL lane its consumers call at a release tag: Analyze per language with build-mode none and Default Setup's /language:<language> category, least-privilege permissions per job, and the engine's extraction guard, fetched at the workflow's own commit and run from runner.temp. Skills adopts the lane on itself, so a fork pull request gets a CodeQL run without Default Setup.

* fix(ci): quote the CodeQL lane's languages description (#256)

An unquoted 'build-mode: none' inside the input description is a YAML mapping, so GitHub refused the workflow file.

* ci(codeql): skill-corpus runs the CodeQL suites (#256)

skill-corpus.yml runs each contract suite by name rather than through npm test, so the two suites this PR added never ran in CI. Both now have their own step, and the tarball check asserts that the guard and its two suites stay out of the consumer package.

* fix(codeql): the guard matches only a job's last segment (#256)

ANALYZE_JOB matched an 'Analyze (' anywhere after a segment boundary, so a caller job named 'Analyze (CodeQL)' would have made the guard's own job read as an Analyze leg. The pattern now anchors to the last segment, and the spec pins that caller name."
- 2026-11-12T00:01:39Z @tobiu closed this issue

