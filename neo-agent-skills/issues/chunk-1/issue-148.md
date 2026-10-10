---
id: 148
title: Separate Dependabot validation from Skills releases
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - build
assignees:
  - neo-gpt
createdAt: '2026-10-09T21:03:42Z'
updatedAt: '2026-10-09T23:07:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/148'
author: neo-gpt
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
closedAt: '2026-10-09T23:07:53Z'
---
# Separate Dependabot validation from Skills releases

## Context

[Dependabot PR #147](https://github.com/neomjs/neo-agent-skills/pull/147) changes only Acorn's lock entry at `48be7013d2303ac5b66d6e675c0382214333661b`. [CI run 37907888571](https://github.com/neomjs/neo-agent-skills/actions/runs/37907888571) installs successfully, then refuses unchanged, already-published Skills `0.1.30`; all later contract/corpus checks are skipped.

## The Problem

The current version gate and dev-push publisher treat dependency maintenance as a package release. Exempting only the PR check would still invoke publication after a human merges the bot PR.

## The Architectural Reality

`skill-corpus.yml` invokes `scripts/check-version-bump.mjs` on PRs. `publish.yml` runs tests and publishes/tags on every dev push. GitHub's PR author identifies Dependabot; the workflow actor may instead be the human merging or rerunning it. The commit-associated PR API returns a merged PR with `merge_commit_sha`, base repository/ref and author; verified on dev commit `207670e83e680c1118cb7d71133e10a87e1e643e` and PR #146.

Design authority: operator correction on 2026-10-09 — “our own PRs should always update the package version and publish. dependabot PRs should NOT do this.” This supersedes #56's every-PR release scope for Dependabot only.

## The Fix

Use one exact Dependabot-author predicate for version validation and release eligibility. Dependabot PRs retain the base package version and matching lock metadata, still run the complete validation suite, and never enter the publish/tag job after merge. Other PRs retain the greater/unpublished version rule and publish normally. Post-merge classification uses the exact merged PR for the commit and base, never actor, branch name or commit text. Unavailable or ambiguous origin fails closed. Document the boundary and add source-workflow/behavior controls using the existing `scripts/` guard/test pattern.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs / evidence |
|---|---|---|---|---|
| PR version check | GitHub PR author + base package version | Dependabot unchanged; other authors new unpublished release | unreadable event/base/registry fails where required | README; version contract |
| Publish eligibility | Exact merged PR associated with commit, repository and dev base | Dependabot no publish/tag; other merged PRs publish | missing/ambiguous origin fails | README; human-merge and wrong-origin controls |
| Validation selection | Skills corpus and publish test workflows | Both authors run tests | failed tests block release | workflow wiring + full suite |
| Release serialization | Eligible publish job after exact origin classification | `queue: max`, non-cancelling; bot-only runs never enter | queued releases preserved up to the platform limit | README; workflow concurrency contract |

Decision Record impact: none; localized release-policy change at explicit operator direction.

## Acceptance Criteria

- [ ] AC-1: Dependabot's unchanged release version passes without an npm lookup; a bot package-version bump fails. Other authors keep greater/unpublished version and lock agreement checks.
- [ ] AC-2: A Dependabot PR merged or rerun by a human cannot publish or tag. Maintainer PRs remain release-eligible. Unknown, unmerged, wrong-base and ambiguous commit origins cannot grant publication.
- [ ] AC-3: Production workflows invoke these paths and retain corpus/contract tests for Dependabot. The policy implementation PR itself carries a fresh version.
- [ ] AC-4: README states maintainer-release versus Dependabot-validation behavior; deterministic controls and the full source test suite pass.

## Post-Merge Validation

Observe #147 rebased onto the landed policy passing its unchanged-version gate, and its eventual human merge skipping publication/tagging. Observe this policy PR's own release publish/tag. Hosted post-merge receipts remain integration follow-through under #14; local/API controls do not claim an npm publication.

## Out of Scope

No Acorn code change, package release from #147, consumer scheduling change, auto-merge, credential change or exception for other bot accounts.

## Avoided Traps

Checking `github.actor` would misclassify a human merging Dependabot. Skipping only the bump check leaves the publish path wrong. Trusting branch/commit names allows a maintainer PR to inherit the exception. No new release framework or duplicated publisher.

## Related

#147 · #56 · #14. #144 governs downstream propagation, a separate outcome.

Creation checks: latest 20 live open issues and sole open PR read immediately before filing; no equivalent. All-state recent A2A claims checked; no overlap. Memory Core rationale searches surfaced prior version policy, not an existing exception; KB returned unrelated/partial references. Own-assignment sweep found #140, body read: recipient-load integration, not this release gate. Structural fast-path: new source-only release guard and test match `scripts/check-version-bump.mjs`, `scripts/tag-release.mjs` and their contract runners; no `ai/` or skill-substrate touch.

Origin Session ID: 1690d62c-24ed-41e2-93e0-22159beeb56f
Retrieval Hint: Skills Dependabot unchanged version corpus gate merged PR publish identity

## Timeline

- 2026-10-09T21:03:42Z @neo-gpt assigned to @neo-gpt
- 2026-10-09T21:03:44Z @neo-gpt added the `enhancement` label
- 2026-10-09T21:03:45Z @neo-gpt added the `ai` label
- 2026-10-09T21:03:45Z @neo-gpt added the `testing` label
- 2026-10-09T21:03:45Z @neo-gpt added the `build` label
- 2026-10-09T21:04:22Z @neo-gpt cross-referenced by PR #147
- 2026-10-09T21:15:03Z @neo-gpt referenced in commit `03e403d` - "feat(release): keep Dependabot validation-only (#148)"
- 2026-10-09T21:15:05Z @neo-gpt cross-referenced by PR #149
- 2026-10-09T21:43:51Z @neo-fable-clio cross-referenced by #150
- 2026-10-09T22:10:28Z @neo-gpt referenced in commit `01e40b7` - "docs(release): state dependency delivery timing (#148)"
- 2026-10-09T23:07:53Z @tobiu referenced in commit `47ab9eb` - "feat(release): keep Dependabot validation-only (#148) (#149)

* feat(release): keep Dependabot validation-only (#148)

* docs(release): state dependency delivery timing (#148)"
- 2026-10-09T23:07:53Z @tobiu closed this issue
- 2026-10-10T00:29:30Z @neo-fable-clio cross-referenced by #154

