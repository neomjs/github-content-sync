---
id: 285
title: Commit messages can close a ticket the PR does not resolve
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - model-experience
assignees:
  - neo-opus-ada
createdAt: '2026-11-06T00:01:21Z'
updatedAt: '2026-11-06T11:45:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/285'
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
closedAt: '2026-11-06T11:45:33Z'
---
# Commit messages can close a ticket the PR does not resolve

## Context

A closing expression in a commit message closes its target when the commit reaches the default branch. No guard reads commit messages for one.

- **2026-05, measured:** `neomjs/neo#11183` was squash-merged as `5c7c5a2f4`. A branch commit body still carried a closing expression for `#11182`, which the corrected PR body only referenced, so `#11182` auto-closed and had to be reopened by hand. neomjs/neo#11185 responded with discipline, not a guard: an author pre-handoff `git log origin/dev..HEAD` scan in `pull-request-workflow.md`, and a reviewer commit-message parse in `pr-review-guide.md` §5.2.
- **2026-11-05, near-miss:** in `neomjs/neo#20402`, my commit subject put the noun "fix" directly before the number of another open ticket (`#20403`). Nobody ran the documented scan.
  - `baseline / PR body` caught the copy of the phrase in the PR body.
  - `baseline / Commit authorship` passed on that head (`029fc2ba33`), and so did the pre-push run of the same guard.
  - With a clean body, the PR would have gone green. The documented clean path, a fresh branch, became `neomjs/neo#20405`.

Every merge method on neo carries branch commit messages to `dev`. `gh api repos/neomjs/neo` reports `default_branch: dev` and `squash_merge_commit_message: COMMIT_MESSAGES`, and merge commits and rebase merges are enabled too.

## The Problem

The one-close-target rule judges exclusivity "over everything GitHub acts on" (`findCloseTargetViolations`' docblock), but it is enforced only on the PR body. Commit messages are the second channel, and two manual steps are their only guard. A busy author or reviewer skips them.

"Fix" and "resolve" are ordinary English (a noun, a quotation, "resolves a conflict"), so the pair gets written by accident. The damage is silent: per GitHub's docs, a commit-based close does not list the PR as linked, and the ticket has to be reopened by hand.

## The Architectural Reality

Anchors at `neo-agent-skills` `dev` `2a17d17`:
- `scripts/check-pr-body.mjs:152` exports `GITHUB_CLOSING_EXPRESSION`: any case, an optional colon, and a `#N`, `owner/repo#N` or issue-URL target, per GitHub's docs. `findCloseTargetViolations` (`:170`) applies it to the PR body only.
- `scripts/check-commit-authorship.mjs` (bin `neo-agent-skills-commit-authorship`) is the one guard that reads commits, and it runs at both ends:
  - `pendingRanges` (`:57`) picks the commits a push introduces. It uses `--not --remotes`, so upstream history is never re-judged; in CI it uses `<base>..HEAD`.
  - `readCommits` (`:251`) returns `{sha, authorEmail, subject, body}`.
  - `run` (`:297`) runs two checks: operator identity and unknown co-authors.
- Consumers run it before every push (neo's `.husky/pre-push`) and in CI (`.github/workflows/reusable-pr-baseline.yml:293`, `Commit authorship`, `--base <PR base sha>`).

## The Fix

Add a third check to the commit-range guard: no commit message, subject or body, may carry a closing expression. The PR body's `Resolves #N` stays the one place a PR declares its close.
- A refusal names the commit, the expression and the rule. At pre-push, the remedy is an amend before the push.
- Import `GITHUB_CLOSING_EXPRESSION` from `check-pr-body.mjs`, so one matcher covers both channels.
- Widen the guard's `@summary` (`:3`) to name the third refusal.
- Specs go in `scripts/test-commit-authorship.mjs`.
- Once the guard exists, the two manual scans shrink to a pointer to it.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-skills-commit-authorship`: exit code and message, at pre-push and in the `--base` CI run | the one-close-target rule (`findCloseTargetViolations` docblock; @tobiu 2026-09-21 / 2026-09-23); neomjs/neo#11185's measured auto-close | exit 1 when a commit in the range carries any closing expression; the message names the SHA, the expression and the rule | commits a remote already has are never judged (`pendingRanges`); `fix(scope):` prefixes and a bare `(#N)` are not closing expressions | the guard's `@summary`; the two skill sections AC-6 shrinks | AC specs |

## Acceptance Criteria

- [ ] AC-1: The guard refuses a commit whose subject or body carries any expression `GITHUB_CLOSING_EXPRESSION` matches, including a past-tense one inside prose. It exits 1 and names the commit SHA, the expression and the rule.
- [ ] AC-2: These pass: a `fix(scope):` or `fix:` subject prefix with no target after it, a bare `(#N)`, and `Refs #N`.
- [ ] AC-3: Both entry points enforce it, with a spec arm each: a pre-push ref-tuple range and a CI `--base` range. Upstream commits a branch merges in are not re-judged.
- [ ] AC-4: Red on base, green at head: fixtures shaped like the two incidents pass today's guard and fail the new one. One is a subject with "fix" as a noun before another ticket's number. The other is a body with a stale `Resolves #N`.
- [ ] AC-5: One matcher. The guard imports `GITHUB_CLOSING_EXPRESSION`, and no second closing regex exists in `scripts/`.
- [ ] AC-6: The pre-handoff scan in `pull-request-workflow.md` and step 1 of `pr-review-guide.md` §5.2 point at the guard instead of restating the manual check. Net loaded bytes fall.

## Out of Scope

- Commits already on `dev` or any other remote branch.
- The PR-body lint, which is unchanged.
- Repository merge settings (neomjs/neo#11185 excluded them too).
- Rewriting commits: the guard only refuses.

## Avoided Traps

- **Exempting the subject's own `(#N)` ticket.** It keeps a harmless duplicate legal, but it lets a commit carried over from another ticket's branch close that ticket. The PR body is the one declaration.
- **A separate `check-commit-messages.mjs`.** A second reader of the commit range, plus new hook and workflow wiring in every consumer, for one rule.
- **A CI-only comparison against the PR body's target.** It catches the problem after the push and couples the commit guard to the body. A pre-push refusal is cheaper, because amending an unpushed commit is allowed.
- **Switching `squash_merge_commit_message`.** It covers squash merges only, and it is a per-repository setting, not a guard.

## Related

- neomjs/neo#11185: the measured auto-close and its discipline-only fix, which this makes mechanical.
- `scripts/check-pr-body.mjs`'s close-target rule: the body channel.

Design authority: the one-close-target rule in `findCloseTargetViolations`' docblock (`scripts/check-pr-body.mjs:156`–`:169`) judges exclusivity "over everything GitHub acts on". The `#11182` auto-close that neomjs/neo#11185 measured shows that GitHub acts on commit messages too.
Decision Record impact: none.
Structure map gate (§1c): N/A. This touches `neo-agent-skills` `scripts/` CI guards, not Brain `ai/`, MCP or placement, and the existing commit-range guard owns it.
Meta-skill sweep (§1b): AC-6 shrinks two reference payloads, never a `SKILL.md` router.
Ownership: self-assigned. It is not a good first issue: I'm landing it now, and the dual hook/CI entry needs the guard's internals.

Live latest-open + closed sweep: the latest 20 open here, closed searches ("commit message closing", "close target") and an org search, last run at 2026-11-06T00:00:39Z. I read Out of Scope / Avoided Traps on `#103` and neomjs/neo#11185 and found no equivalent. neomjs/neo#11185's exclusions are merge settings and commit rewriting, not a detection guard. A2A in-flight sweep (latest 30, all read states): no claim on this scope. MC sweep: a `query_raw_memories` search on the symptom surfaced the `#11182` auto-close. Own-assignment sweep: my open tickets here (`#282`, `#102`) cover other surfaces.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868
Retrieval Hint: "a closing expression in a commit message reaches dev through every merge method; the commit-authorship guard reads the range but not the keyword (neo #11185 discipline → mechanical guard)"

## Timeline

- 2026-11-06T00:01:21Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-06T00:01:23Z @neo-opus-ada added the `enhancement` label
- 2026-11-06T00:01:23Z @neo-opus-ada added the `ai` label
- 2026-11-06T00:01:23Z @neo-opus-ada added the `build` label
- 2026-11-06T00:01:23Z @neo-opus-ada added the `model-experience` label
- 2026-11-06T00:34:23Z @neo-opus-ada cross-referenced by PR #286
- 2026-11-06T01:02:26Z @neo-opus-ada referenced in commit `3d7edcd` - "fix(guard): read whole commit messages and judge every author's text (#285)

Round 1 (Euclid): the closing check exempted a GitHub App PR author, and readCommits split messages at its in-band \x1f/\x1e delimiters, so an expression after either byte went unseen. The rule now reads the published text alone, and git log -z with newline-separated fields reads every message whole. Both arms were red before their repair."
- 2026-11-06T11:45:33Z @tobiu referenced in commit `4360418` - "feat(guard): refuse closing keywords in commit messages (#285) (#286)

* feat(guard): refuse closing keywords in commit messages (#285)

The commit-authorship guard gains a third check, at pre-push and in CI, for every author except a GitHub App's PR: no commit message carries a closing expression, judged by GITHUB_CLOSING_EXPRESSION, the matcher the PR body already answers to. Squash bodies built from commit messages carry such a phrase to the default branch, where it closes the issue: measured on neo in May (#11185), nearly repeated on neo #20402. The two manual scans in pull-request-workflow and pr-review-guide shrink to a pointer (net -282 skill bytes).

* fix(guard): read whole commit messages and judge every author's text (#285)

Round 1 (Euclid): the closing check exempted a GitHub App PR author, and readCommits split messages at its in-band \x1f/\x1e delimiters, so an expression after either byte went unseen. The rule now reads the published text alone, and git log -z with newline-separated fields reads every message whole. Both arms were red before their repair."
- 2026-11-06T11:45:34Z @tobiu closed this issue

