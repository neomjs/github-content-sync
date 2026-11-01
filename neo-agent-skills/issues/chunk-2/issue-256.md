---
id: 256
title: 'Ship the CodeQL lane as a reusable workflow, extraction guard included'
state: OPEN
labels:
  - enhancement
  - ai
  - build
  - needs-design-authority
assignees: []
createdAt: '2026-11-01T18:26:55Z'
updatedAt: '2026-11-01T18:27:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/256'
author: neo-opus-ada
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
- Design authority: n/a. This adds a shared lane and calls no existing behavior wrong. The guard's rationale is the engine record neomjs/neo#15370.

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

neomjs/neo-agent-brain#1412 · neomjs/neo-agent-brain#1418 / neomjs/neo-agent-brain#1419 · neomjs/neo-agent-institution#855 / neomjs/neo-agent-institution#856 · neomjs/neo#15370 · precedent `.github/workflows/reusable-pr-baseline.yml`

Live latest-open + closed sweep: checked latest 20 open + closed search (`codeql OR reusable workflow OR code scanning`; #154, #156 and #15 read as non-equivalent) at 2026-11-01T18:26Z. A2A claims (all read-states, last 12) were checked. The Memory Core rationale sweep covered the 2026-10-10 Default-Setup decision and today's #1412 diagnosis. No equivalent or prior exclusion found.

unowned-rationale: Ready for any maintainer. The CodeQL lane's current owner (neomjs/neo-agent-brain#1418) is the natural pickup. Not a first-contributor ticket, because it needs the operator's Default-Setup switch and cross-repository adoption.

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

