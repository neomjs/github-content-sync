---
id: 23
title: Allow human release PRs in the shared base guard
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - testing
  - build
assignees: []
createdAt: '2026-08-30T18:04:42Z'
updatedAt: '2026-08-30T18:04:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/23'
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
# Allow human release PRs in the shared base guard

Parent Epic: #14

## Context

#15 / PR #17 published the first reusable PR baseline with a stable `pr-base` job. Its contract accepts one `required_base` input (default `dev`) and fails every caller PR whose base differs. That is correct for agent-authored development PRs and is already consumed by DevIndex.

Engine's live release contract is broader in one deliberate direction: `.github/workflows/pr-base-guard.yml` permits the human release authority `tobiu` to target `main`, while unauthorized authors are retargeted/refused. The `pull-request` skill and repository critical gates encode the same distinction: agent-authored PRs target `dev`; `main` is release-only and human-directed.

The current reusable job has no actor/release exception. Adopting it unchanged in Engine or Brain would make an authorized human `main` release PR red. Passing the observed base back as `required_base` would make the guard tautological and silently admit agent `main` PRs.

Measured at Skills `dev@d6551c8a04`; consumer census and publication state are recorded on Epic #14 comment `5470359432`.

## The Problem

- The reusable baseline cannot represent the organization's two legitimate base outcomes: ordinary agent PR → `dev`; authorized release PR → `main`.
- The only available caller workaround either rejects valid releases or weakens the check into `actual == actual`.
- Engine cannot delete its local base guard while this semantic gap exists, so broad baseline adoption would add duplicate jobs rather than retire duplicated governance.
- Actor authority is security-sensitive: an empty/malformed allowlist must fail closed, not widen release admission.

## The Architectural Reality

- The reusable workflow owns policy execution; callers own only immutable source coordinate and explicit repository inputs.
- `pull_request` payload already provides both base ref and author login. No checkout, write permission, or GitHub API call is required.
- Release authority is repository policy, so the reusable workflow should expose explicit inputs rather than hardcode a growing organization roster.
- The stable job id/name must not change; repository required-context bindings depend on it.

## The Fix

Extend `.github/workflows/reusable-pr-baseline.yml` with an explicit release exception:

1. `release_base` string input, default `main`.
2. `release_actors` JSON-array string input, default `[]` (no release exception unless the caller opts in).
3. The base job admits exactly one of:
   - `pull_request.base.ref == required_base`; or
   - `pull_request.base.ref == release_base` and the exact author login is present in the parsed release actor set.
4. Missing PR payload, invalid JSON, non-string actor entries, empty actor identity, wrong base, or unauthorized release actor fails closed with a bounded diagnostic.

Extend the existing reusable-workflow contract suite rather than creating a second test mechanism.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `required_base` | Agent PR target rule | Ordinary PRs must target the configured development branch | Mismatch is red | Workflow input description | dev positive + wrong-base negative fixtures |
| `release_base` | Human-only release rule | Names the protected release branch eligible for the actor exception | Empty or same-as-required value does not widen admission | Workflow input description | main-release fixtures |
| `release_actors` | Repository caller policy | Exact JSON string array of identities allowed to use `release_base` | Invalid shape/value fails closed; default `[]` permits nobody | Workflow comments + caller example | parser mutations + unauthorized actor witness |
| `pr-base` status | Epic #14 stable-context contract | Keeps the existing job id/name while evaluating both legitimate branches | No additional status context | Existing workflow docs | contract test pins id/name |

## Decision Record impact

`none` — this completes the existing human-release/agent-development distinction and changes no ADR or product runtime.

## Acceptance Criteria

- [ ] `reusable-pr-baseline.yml` accepts explicit `release_base` and JSON-array `release_actors` inputs while preserving existing defaults for ordinary `dev` PRs.
- [ ] The `pr-base` job admits an ordinary required-base PR and an authorized release-base PR, and rejects an agent/unauthorized release-base PR.
- [ ] Missing/non-PR payload, invalid JSON, non-array input, non-string/empty actor entries, and unsupported base all fail closed with bounded diagnostics.
- [ ] The existing `pr-base` job id and `PR base` display name remain unchanged.
- [ ] Workflow permissions remain read-only; the exception performs no retarget, comment, checkout, or GitHub mutation.
- [ ] Mutation-sensitive tests cover both positive paths and every fail-closed input/base/actor branch.
- [ ] Existing skills-materialization and source-comment-archaeology jobs remain unchanged and green.
- [ ] No consumer caller is changed by this source PR; Engine/Brain callers become separate leaves after this contract lands.

## Out of Scope

- Changing who holds human release authority in any repository.
- Retargeting or closing invalid PRs; this reusable job is a status check only.
- Required-status repository settings.
- Consumer caller rollout, package publication, or workflow pin updates.
- PR-review-body policy (#22) or hook transport (#21).

## Avoided Traps

- Passing `github.event.pull_request.base.ref` back as the expected base.
- A mutable/hardcoded organization-wide release roster inside the shared workflow.
- Comma-splitting an unvalidated string where whitespace or empty entries widen admission.
- Renaming the stable job/context during a semantic extension.
- Keeping Engine's local guard indefinitely and adding the shared guard beside it.

## Related

- Parent: #14
- Source baseline: #15 / PR #17
- Consumer/release census: #14 comment `5470359432`
- PR-review policy sibling: #22
- Hook transport sibling: #21

Origin Session ID: 96f8b385-2f2e-4730-8185-36b7fb18f9f4

Retrieval Hint: "shared PR baseline human main release actor exception required_base tautology"
Retrieval Hint: Skills `reusable-pr-baseline.yml@d6551c8a04`; Engine `.github/workflows/pr-base-guard.yml@7ecd6b8b6e`


## Timeline

- 2026-08-30T18:04:43Z @neo-gpt-emmy added the `bug` label
- 2026-08-30T18:04:43Z @neo-gpt-emmy added the `ai` label
- 2026-08-30T18:04:44Z @neo-gpt-emmy added the `architecture` label
- 2026-08-30T18:04:44Z @neo-gpt-emmy added the `testing` label
- 2026-08-30T18:04:44Z @neo-gpt-emmy added the `build` label
- 2026-08-30T18:04:48Z @neo-gpt-emmy added parent issue #14
- 2026-08-31T06:45:44Z @neo-opus-grace cross-referenced by #25
- 2026-09-01T22:46:31Z @neo-fable cross-referenced by #38
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

