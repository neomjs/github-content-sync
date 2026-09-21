---
id: 22
title: Publish reusable agent PR-review policy
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - testing
  - build
  - model-experience
assignees: []
createdAt: '2026-08-30T17:53:58Z'
updatedAt: '2026-08-30T17:53:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/22'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 14
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
# Publish reusable agent PR-review policy

Parent Epic: #14

## Context

Epic #14 selected `neo-agent-skills` as the source of truth for non-product PR governance, with repository-local callers and Brain retaining only managed GitHub mutation/orchestration. The first two source leaves proved the transport: #15 / PR #17 published the reusable PR baseline, and #18 / PR #19 added a packaged caller-repository CLI plus a third stable job.

The next policy surface is already duplicated. At Engine `dev@7ecd6b8b6e`, `.github/workflows/agent-pr-review-body-lint.yml` is a 605-line, 33,399-byte inline post-submit validator. At Brain `dev@90d41ff90a`, `ai/services/github-workflow/PullRequestService.mjs` owns the managed pre-submit contract and explicitly records that CI and service copies can disagree. The Engine workflow hardcodes the same review shapes, state coherence, demand detection, terminal Drop+Supersede contract, review-budget provenance, and activation coordinate.

This is not the 2026-05 same-checkout extraction from `neomjs/neo#11501`. That attempt imported Engine-local `ai/services/**` modules from Engine-local workflows and was reverted in PR `neomjs/neo#11502`; sync-by-convention was accepted for the then-single repository. The post-split authority is different: the policy templates now live in this package, five repositories consume it, and Epic #14 explicitly replaced copy-per-repository governance with a canonical package/workflow source.

Structure-map gate: Brain `npm run --silent ai:structure-map -- --files --loc` confirms the managed GitHub service remains under `ai/services/github-workflow`; this leaf adds no Brain runtime source. Structural fast-path: `scripts/check-pr-review-body.mjs` follows the published CLI pattern established by `scripts/check-ticket-archaeology.mjs`, paired with a mutation-sensitive `scripts/test-*.mjs` contract and a reusable workflow under `.github/workflows`.

## The Problem

- A review policy edit currently requires synchronized changes in two repositories and two execution models, with no single test able to prove parity.
- The Engine workflow is both policy and transport: 605 lines of inline JavaScript inside YAML are not directly reusable by Brain or testable as a package contract.
- Brain's retained suite still reaches the Engine workflow to test policy parity. After the repository split removes the Engine projection, those checks have no canonical file to read.
- Institution, DevIndex, Brain, and Skills have no equivalent `pull_request_review` caller, so direct `gh pr review` / UI submissions receive different enforcement depending on repository.
- The workflow is post-submit telemetry while `manage_pr_review` is pre-submit mutation control. Flattening them into one implementation would erase a real service boundary; letting their deterministic policy diverge is equally wrong.

## The Architectural Reality

- Package-owned templates under `.agents/skills/pr-review/**` are the policy inputs.
- A deterministic module can validate body/state shapes without knowing GitHub transport. It receives resolved facts; it does not fetch PR history or post comments.
- Brain continues to own managed review creation/update, reviewer-family budgeting, and GitHub GraphQL orchestration. A later consumer leaf imports the package policy instead of reimplementing it.
- A separate reusable workflow must own `pull_request_review` semantics. It must not be folded into `reusable-pr-baseline.yml`, whose `pull_request` event and base/materialization jobs intentionally have different semantics.
- The reusable workflow wrapper owns event acquisition, bounded GitHub reads, and corrective-comment side effects. It invokes the package validator and exposes one stable job name.
- Consumer repositories own only the `pull_request_review` trigger, explicit least-privilege grant, and immutable Skills release coordinate.

## The Fix

1. Add `scripts/check-pr-review-body.mjs` as the published policy module/CLI. Export deterministic validation functions through a `./pr-review-policy` package subpath and expose a bin for workflow execution.
2. Derive the supported review shapes from the package-local `pr-review` assets, while keeping invisible enforcement details absent from diagnostics.
3. Add `.github/workflows/reusable-pr-review.yml` with `workflow_call`, an independently stable job name, explicit least-privilege permissions, exact caller event/head coordinates, and the post-submit corrective signal.
4. Add mutation-sensitive module, CLI, workflow, and package-boundary tests. Every policy branch carried from the current Engine workflow must have a positive fixture and a red mutation.
5. Keep consumer callers and Brain adoption out of this source PR; they become separate Epic leaves after the canonical source lands.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-skills/pr-review-policy` | Package `pr-review` assets + Epic #14 | Exports deterministic full, micro, micro-delta, Round-2, terminal, demand, and provenance validation | Unknown shape or contradictory state fails closed with a named code; invisible anchors never enter diagnostics | Module JSDoc + README API note | Fixture matrix + one mutation per branch |
| `neo-agent-skills-validate-pr-review` | Same policy module | Validates one normalized review/event envelope and emits machine-readable result + exit code | Malformed/missing envelope exits 2; policy violation exits 1; valid/irrelevant human review exits 0 | CLI `--help` | Spawned CLI tests |
| `.github/workflows/reusable-pr-review.yml` | GitHub `workflow_call` + package policy | Runs against the caller's exact `pull_request_review` event and exposes a stable status name | Non-review invocation, missing coordinates, incomplete activation history, or policy failure is red; no silent skip | Workflow comments | Workflow contract tests + negative mutations |
| Corrective comment | Reusable workflow wrapper | Posts the existing skill-pointing remediation without disclosing invisible anchors | Duplicate edit events remain idempotent or update one marker-owned comment | Workflow comments | Mocked wrapper fixture / rendered message assertions |
| Published package | `package.json#files`, `exports`, and `bin` | Ships the policy module/CLI and required template assets; excludes workflow/test implementation | Missing export/bin or leaked `.github`/test sources is red | README distribution section | `npm pack` contract |
| Consumer adoption | Separate leaves under Epic #14 | Minimal immutable caller per repository; Brain imports the same policy in its managed service | No consumer is edited by this source PR | Epic relationship graph | Later consumer PRs |

## Decision Record impact

`none` — this is a source leaf of Epic #14 and changes no review policy or runtime ADR. It replaces the pre-split sync-by-convention transport with the package authority already selected for the five-repository topology; it does not resurrect checkout-local cross-imports from the reverted `neomjs/neo#11501` shape.

## Acceptance Criteria

- [ ] A published `./pr-review-policy` subpath and CLI own the deterministic review-body/state contract without importing Brain or Engine runtime modules.
- [ ] The policy covers canonical full reviews, micro reviews, micro-delta closure, ordinary Round 2, Drop+Supersede, COMMENT demand detection, origin-session shape, and review-budget provenance carried by the current managed/workflow surfaces.
- [ ] Machine results use stable named codes and never disclose invisible anchor strings; valid human/non-agent reviews remain an explicit non-enforced outcome rather than an accidental skip.
- [ ] A separate least-privilege `workflow_call` workflow consumes the package policy for `pull_request_review`, operates on the caller repository/event, and exposes a stable job name.
- [ ] The wrapper preserves bounded activation-history reads and one corrective, skill-pointing comment without moving managed review mutation into Skills.
- [ ] Mutation-sensitive tests red each policy branch plus trigger, permissions, caller coordinate, immutable package invocation, comment privacy, and non-review fail-closed behavior.
- [ ] `npm pack` contains the policy export/bin and templates but excludes `.github/**` and source-only contract tests.
- [ ] Existing `reusable-pr-baseline.yml`, source-comment archaeology, and materializer contracts remain green.
- [ ] No consumer repository is modified by this PR; caller rollout and Brain adoption remain separate Epic leaves.

## Out of Scope

- Rewriting the review policy, templates, one-round budget, cross-family mandate, or reviewer identity model.
- Moving `manage_pr_review`, GitHub mutation services, GraphQL acquisition, or Memory Core ingestion into Skills.
- Combining `pull_request` and `pull_request_review` event semantics in one reusable workflow.
- Consumer caller files, required-status settings, or deletion of the Engine workflow.
- Hook transport; #21 owns that runtime sibling.

## Avoided Traps

- Importing a caller checkout's `ai/services/**` from workflow YAML, the reverted pre-split shape.
- Copying the 605-line workflow into four more repositories and calling the copies parity.
- Publishing only anchor arrays while leaving Round-2, terminal, demand, and budget semantics duplicated.
- Hiding GitHub side effects inside the pure policy module, which would make Brain/runtime reuse non-deterministic.
- Folding review-event CI into the PR baseline and making unrelated jobs fail on the wrong event type.
- Shipping the reusable source without explicit later consumer leaves.

## Related

- Parent: #14
- Reusable workflow precedent: #15 / PR #17
- Packaged guard precedent: #18 / PR #19
- Hook transport sibling: #21
- Historical trap: `neomjs/neo#11501` and PR `neomjs/neo#11502`
- Brain retained-suite owners: `neomjs/neo-agent-brain#194` and `neomjs/neo-agent-brain#201`

Origin Session ID: 96f8b385-2f2e-4730-8185-36b7fb18f9f4

Retrieval Hint: "reusable pull_request_review policy package validator Engine workflow Brain PullRequestService"
Retrieval Hint: Engine `.github/workflows/agent-pr-review-body-lint.yml` at `7ecd6b8b6e`; Brain `ai/services/github-workflow/PullRequestService.mjs` at `90d41ff90a`; Skills `reusable-pr-baseline.yml` at `d6551c8a04`


## Timeline

- 2026-08-30T17:54:00Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-30T17:54:00Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T17:54:00Z @neo-gpt-emmy added the `architecture` label
- 2026-08-30T17:54:00Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T17:54:00Z @neo-gpt-emmy added the `build` label
- 2026-08-30T17:54:01Z @neo-gpt-emmy added the `model-experience` label
- 2026-08-30T17:54:08Z @neo-gpt-emmy added parent issue #14
- 2026-08-30T18:03:13Z @neo-gpt-emmy cross-referenced by #14
- 2026-08-30T18:04:43Z @neo-gpt-emmy cross-referenced by #23
- 2026-08-30T21:12:53Z @neo-gpt-emmy cross-referenced by #64
- 2026-08-31T06:45:44Z @neo-opus-grace cross-referenced by #25
- 2026-08-31T06:53:27Z @neo-opus-grace cross-referenced by #17913
- 2026-08-31T06:53:44Z @neo-opus-grace cross-referenced by PR #17915
- 2026-08-31T07:37:01Z @neo-opus-grace cross-referenced by #28
- 2026-08-31T08:07:23Z @neo-opus-grace cross-referenced by #29
- 2026-08-31T09:47:34Z @neo-gpt-emmy cross-referenced by PR #30
- 2026-09-01T22:46:31Z @neo-fable cross-referenced by #38

