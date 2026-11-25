---
id: 470
title: Wire test-claude-plugin and test-agents-md-pointer into skill-corpus CI
state: CLOSED
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - testing
  - build
assignees:
  - Rewaskc02-lang
createdAt: '2026-11-25T17:03:54Z'
updatedAt: '2026-11-25T19:18:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/470'
author: neo-gemini-julia
commentsCount: 2
parentIssue: 446
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-25T19:18:41Z'
---
# Wire test-claude-plugin and test-agents-md-pointer into skill-corpus CI

Please comment to claim before starting; check current assignees and linked PRs.

## Context

Promoted from `@neo-opus-ada`'s defect-note (`MESSAGE:d2a08150-b1b4-47e9-a70a-17f38b5d1843`, `2026-11-25T16:50Z`) after two consecutive merges (#465 and #466) landed stale `plugins/**` / `.claude-plugin/marketplace.json` projections on `dev` without CI failing.

## The Problem

`package.json` (`scripts.test`, line 69) chains 25 contract suites, including `node scripts/test-claude-plugin.mjs` (which runs `generate-claude-plugin.mjs --check`, manifest bounds, Policy §2F fetchable-URL bans, and #467's relative-link resolution guard) and `node scripts/test-agents-md-pointer.mjs`.

However, `.github/workflows/skill-corpus.yml` (`corpus` job, lines 49–128) invokes contract suites **individually by filename** rather than running `npm test` — and neither `node scripts/test-claude-plugin.mjs` nor `node scripts/test-agents-md-pointer.mjs` was added to `skill-corpus.yml`! As the comment at `.github/workflows/skill-corpus.yml:90-92` already warns (`"because this job runs each suite BY NAME rather than through npm test: a contract added to the package script alone would still never be selected here"`), omitting a suite from `skill-corpus.yml` leaves its entire contract unexecuted on pull requests and `dev` pushes:
- #465 merged without projecting #463's skill changes into `plugins/adversarial-pr-review`.
- #465 and #466 bumped `package.json` (`2.8.6`, `2.8.7`) via manual edits instead of `npm version patch`, leaving `.claude-plugin/marketplace.json` and `plugins/*/.claude-plugin/plugin.json` at `2.8.5` while CI reported green.

## The Architectural Reality

- `.github/workflows/skill-corpus.yml`:
  - `corpus` job (lines 49–128) runs 23 `scripts/test-*.mjs` suites individually, omitting `scripts/test-claude-plugin.mjs` and `scripts/test-agents-md-pointer.mjs`.
  - `Package contents — what actually ships` step (line 167 `for unwanted in ...`) also omits `package/scripts/generate-claude-plugin.mjs`, `package/scripts/test-claude-plugin.mjs`, and `package/scripts/test-agents-md-pointer.mjs` from its explicit source-only exclusion assertions (currently safe only because `package.json#files` is an allowlist, unlike `package/scripts/agents-md-pointer.mjs` which intentionally ships).
- `package.json` (`scripts.test`, line 69): lists all 25 `node scripts/test-*.mjs` suites.

Design authority: `.github/workflows/skill-corpus.yml:90-92` ("Wired HERE as well as in `package.json#test`, because this job runs each suite BY NAME rather than through `npm test`: a contract added to the package script alone would still never be selected here, which is exactly how the PR-body contract shipped unexecuted.") and Epic #446 Terminal Predicate ("pass automated `--check` and Directory Policy §2D/§2F/§4C checks in `npm test`").

## The Fix

1. Add explicit steps in `.github/workflows/skill-corpus.yml` (`corpus` job) for `node scripts/test-agents-md-pointer.mjs` and `node scripts/test-claude-plugin.mjs` (plus `package/scripts/generate-claude-plugin.mjs`, `package/scripts/test-claude-plugin.mjs`, and `package/scripts/test-agents-md-pointer.mjs` in line 167's `unwanted` tarball check).
2. Add a mechanical parity assertion (in `scripts/test-lint-skill-corpus.mjs` or a dedicated workflow-parity check inside an existing executed `test-*.mjs` suite) verifying that every `scripts/test-*.mjs` referenced in `package.json#scripts.test` is also invoked in `.github/workflows/skill-corpus.yml` (and vice versa), closing the by-name drift class permanently without adding any new GitHub Actions runner job (`CI jobs: +0`).
3. If `origin/dev` carries any projection drift at branch time, run `node scripts/generate-claude-plugin.mjs` alongside the required patch version bump (`npm version patch --no-git-tag-version` on maintainer branches; fork PRs retain base version per `scripts/check-version-bump.mjs`).

## Acceptance Criteria

- [ ] `.github/workflows/skill-corpus.yml` (`corpus` job) runs both `node scripts/test-claude-plugin.mjs` and `node scripts/test-agents-md-pointer.mjs`, and excludes `generate-claude-plugin.mjs`, `test-claude-plugin.mjs`, and `test-agents-md-pointer.mjs` in the tarball `unwanted` loop (`CI jobs: +0`).
- [ ] An automated test executed in `skill-corpus.yml` fails whenever `package.json#scripts.test` and `.github/workflows/skill-corpus.yml` disagree on the set of `scripts/test-*.mjs` suites, verified red against the pre-fix workflow missing `test-claude-plugin.mjs`.
- [ ] `npm test` and `npm run generate-claude-plugin -- --check` exit `0` (with bumped `package.json` / `package-lock.json` / plugin manifests on maintainer PRs, or retained base version on fork PRs per `scripts/check-version-bump.mjs` so the next maintainer release carries the bump).

## Out of Scope

- Adding new GitHub Actions jobs (`CI jobs: +0` — runs inside the existing `corpus` job in ~2 s).
- `claude plugin validate` CLI installation on GitHub runners (already covered structurally by `test-claude-plugin.mjs`).

## Avoided Traps

- Fixing only `test-claude-plugin.mjs` by hand without a `package.json#test` ↔ `skill-corpus.yml` parity assertion: this is already the **third** recurrence (`test-check-pr-body.mjs` per line 92, `test-agents-md-pointer.mjs`, and `test-claude-plugin.mjs`).
- Replacing the named steps in `skill-corpus.yml` with a single opaque `npm test` step if per-contract GitHub UI step annotations are intentionally preserved (either keep named steps + parity test, or verify no workflow contract test pins step names).

## Related

Parent: #446 · Related: #37, #443, #447, #465, #466, #467

---

Live latest-open + closed sweep: checked latest 20 open of 44 (`orange` band; folded as sub-issue under Epic #446) + open/closed searches (`test-claude-plugin`, `generate-claude-plugin`, `skill-corpus`; Out of Scope & Avoided Traps read on #447, #453, #455) at 2026-11-25T16:59Z; #37 covers `npm test` `&&` short-circuiting, not `skill-corpus.yml` suite omission.
A2A in-flight claim sweep: latest 30 all-status messages read at 2026-11-25T17:01Z; promotes `@neo-opus-ada`'s `defect-note` (`MESSAGE:d2a08150-b1b4-47e9-a70a-17f38b5d1843`), zero competing claims.
MC sweep: `"skill-corpus workflow omits test-claude-plugin stale plugin bundle projections npm test"`, 5 results, no prior ticket found.
Own-assignment sweep: 2 open in `neo-agent-skills` (#387, #467), none overlapping.
Structure map (`npm --prefix ../neo-agent-brain run ai:structure-map -- --files --loc`, exit 0): N/A (`neo-agent-skills` `.github/workflows/skill-corpus.yml` + `scripts/`).

unowned-rationale: Ready for a contributor or next free maintainer after #469 merges; maintainers provide planning and review.

Origin Session ID: 05580ab8-2f98-4b02-891c-75d444d212ec
Retrieval Hint: "skill-corpus.yml omits test-claude-plugin test-agents-md-pointer package.json test parity stale plugin projection"

## Timeline

- 2026-11-25T17:03:56Z @neo-gemini-julia added the `bug` label
- 2026-11-25T17:03:56Z @neo-gemini-julia added the `good first issue` label
- 2026-11-25T17:03:57Z @neo-gemini-julia added the `help wanted` label
- 2026-11-25T17:03:57Z @neo-gemini-julia added the `ai` label
- 2026-11-25T17:03:57Z @neo-gemini-julia added the `testing` label
- 2026-11-25T17:03:57Z @neo-gemini-julia added the `build` label
- 2026-11-25T17:04:46Z @neo-gemini-julia added parent issue #446
- 2026-11-25T17:36:10Z @neo-opus-ada cross-referenced by PR #472
### @Rewaskc02-lang - 2026-11-25T17:54:56Z

Hi! I’d like to take this on. I’ll keep the fix within the existing corpus job: add the two missing contract-suite steps, include the related scripts in the tarball exclusion checks, and add a parity test so package.json and the workflow can’t drift again. I’ll run the relevant tests and plugin check before opening a PR. If this is still available, could you assign it to me?

- 2026-11-25T18:10:50Z @neo-gemini-julia assigned to @Rewaskc02-lang
### @neo-gemini-julia - 2026-11-25T18:10:54Z

Welcome @Rewaskc02-lang — assigned! Keeping the two missing contract steps and the `package.json#scripts.test` ↔ `skill-corpus.yml` parity check inside the existing `corpus` job (`CI jobs: +0`) matches the ticket scope cleanly.

One quick heads-up when you run `npm version patch --no-git-tag-version`: open PRs #469 (`2.9.1`) and #472 (`2.9.2`) are currently queued ahead on `dev`, so `node scripts/check-version-bump.mjs` checks against open PR versions as well — feel free to coordinate or rebase once those land.

- 2026-11-25T18:54:13Z @Rewaskc02-lang cross-referenced by PR #473
- 2026-11-25T19:18:40Z @tobiu referenced in commit `ccadd53` - "feat(ci): wire contract suites into corpus CI (#470) (#473)"
- 2026-11-25T19:18:42Z @tobiu closed this issue

