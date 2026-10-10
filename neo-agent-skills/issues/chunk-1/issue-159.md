---
id: 159
title: Dependabot auto-merge covers every Dependabot pull request
state: CLOSED
labels:
  - enhancement
  - ai
  - build
assignees:
  - neo-opus-grace
createdAt: '2026-10-10T16:01:04Z'
updatedAt: '2026-10-10T16:29:41Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/159'
author: neo-opus-grace
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
closedAt: '2026-10-10T16:29:41Z'
---
# Dependabot auto-merge covers every Dependabot pull request

## Context

Design authority: the operator, 2026-10-10 ~16:00Z, asked which Dependabot updates may merge themselves: *"dependabot: ALL PRs once CI is green. no limitations."* This widens the 00:24Z go behind #154 (`neo-agent-skills` bumps only; #154's Out of Scope called widening "its own decision"). The machine-merge class becomes every pull request Dependabot opens, majors included.

0.1.33 (#157) shipped the narrow shape:
- an allow-list (the package and the reusable-baseline tag);
- update types patch and minor;
- the kill switch `NEO_AUTOMERGE_SKILLS`.

No consumer calls it yet, so the defaults and the switch's name can still change for free.

Five repositories run Dependabot (`.github/dependabot.yml`): neo, neo-agent-brain, neo-agent-institution, neo-agent-skills, devindex. pages, github-content-sync and create-app have none.

## The Problem

The operator merges every Dependabot pull request by hand across five repositories, mostly the daily `all-deps` group bumps. The shipped workflow refuses all of them, because none is on its allow-list.

## The Architectural Reality

- `scripts/dependabot-automerge-eligibility.mjs` (`decideAutomergeEligibility`) refuses a dependency outside `allowList` and an update type outside `updateTypes`. A list has no "any" value.
- `.github/workflows/reusable-dependabot-automerge.yml` defaults `dependency_allow_list` and `update_types` to the narrow lists. Its `Read the update` step (`dependabot/fetch-metadata`) fails the job on a pull request it cannot parse, which would stop such a pull request entirely.
- What stays: the author and sender must be `dependabot[bot]`; any other account's push takes the arming back; the live admission (*Allow auto-merge*, required checks, head binding) and `--match-head-commit`. GitHub merges on the base branch's **required** checks, so "CI is green" means what each repository's ruleset requires.
- `agents-md/sections/0401-critical-gate-1.md`'s `machine_merge` line names "allow-listed bumps".

## The Fix

1. Give a list the value `*`, meaning any. With `*` dependencies, an update naming none is still eligible. The workflow defaults both inputs to `*`, and a caller can still pass a list to narrow it.
2. Let the metadata read continue on error, so a Dependabot pull request it cannot parse still reaches the decision. A list-scoped caller refuses it there.
3. Rename the kill switch to `NEO_AUTOMERGE_DEPENDABOT`, since it now covers every Dependabot pull request.
4. Update the gate's `machine_merge` line, the README *Authority* paragraph and the workflow header.
5. Caller leaves follow, one per repository above, under #14.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `dependency_allow_list` / `update_types` inputs | the operator's 16:00Z word | default `*`: any dependency, any update type, majors included | a caller's own list narrows it, as in 0.1.33 | workflow input descriptions | decision fixtures for `*` and for lists |
| `NEO_AUTOMERGE_DEPENDABOT` | rename of the 0.1.33 switch, which has no consumer | `off` stops new armings and takes armed ones back at their next event | none needed: no repository sets the old name | header + README | fixture with the new name; the contract checks the variable read |
| the `machine_merge` line | §critical_gates 1 | names Dependabot's own pull requests on green required checks | — | the generated AGENTS.md | `test-generate-agents-md.mjs`, the byte delta in the PR |

## Acceptance Criteria

- [ ] AC-1 With the defaults, a Dependabot pull request of any dependency, a major bump, and one whose metadata names nothing are eligible; a list still refuses a dependency outside it. The fixtures cover both.
- [ ] AC-2 A failed metadata read no longer fails the job, and the take-back still runs on another account's push (source contract).
- [ ] AC-3 The kill switch is `NEO_AUTOMERGE_DEPENDABOT` in the decision, the workflow and the docs.
- [ ] AC-4 The gate line, README and header describe the new class; the generator contract passes and the PR states the byte delta.
- [ ] AC-5 **Post-merge:** caller leaves in the five repositories (#14). The first auto-merged Dependabot pull request in each is the receipt.

## Out of Scope

The repositories' required-check sets (the operator's ruleset settings); Dependabot's grouping (`dependabot.yml`); pull requests that are not Dependabot's.

## Related

#14 (epic), #154 / #157 (the narrow shape), #158 (0.1.34, merging first).

Sweeps: live latest-open sweep of this repository's 20 newest open issues at 16:00Z, no equivalent (#144 groups Dependabot's skills bumps, which is a different surface) · A2A in-flight sweep, 12 newest messages across read states at 16:01Z: no claim on this · MC sweep: "Dependabot pull requests merge automatically all updates", which surfaces Clio's 00:24Z proposal and the #154 filing, consistent with widening them · Own-assignment sweep: none in this repository.

Origin Session ID: 9ea1c6d0-80b1-4552-b07d-004bb1dba240
Retrieval Hint: "Dependabot auto-merge every pull request allow-list star NEO_AUTOMERGE_DEPENDABOT operator all PRs"

## Timeline

- 2026-10-10T16:01:04Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-10T16:01:05Z @neo-opus-grace added the `enhancement` label
- 2026-10-10T16:01:06Z @neo-opus-grace added the `ai` label
- 2026-10-10T16:01:06Z @neo-opus-grace added the `build` label
- 2026-10-10T16:01:11Z @neo-opus-grace added parent issue #14
- 2026-10-10T16:04:24Z @neo-opus-grace cross-referenced by PR #160
- 2026-10-10T16:04:58Z @neo-opus-grace referenced in commit `a403b76` - "docs(release): the caller block skips pull requests Dependabot did not open (#159)"
- 2026-10-10T16:16:34Z @neo-opus-grace referenced in commit `f737c1e` - "chore(merge): integrate dev's 0.1.34 security release (#159)

# Conflicts:
#	package-lock.json
#	package.json"
- 2026-10-10T16:29:41Z @tobiu closed this issue
- 2026-10-10T16:29:41Z @tobiu referenced in commit `b7b332b` - "feat(release): Dependabot auto-merge covers every Dependabot pull request (#159) (#160)

* feat(release): Dependabot auto-merge covers every Dependabot pull request (#159)

* docs(release): the caller block skips pull requests Dependabot did not open (#159)"
- 2026-10-10T16:32:35Z @neo-opus-grace cross-referenced by #19560
- 2026-10-10T16:32:40Z @neo-opus-grace cross-referenced by #970
- 2026-10-10T16:32:47Z @neo-opus-grace cross-referenced by #661
- 2026-10-10T16:32:54Z @neo-opus-grace cross-referenced by #161
- 2026-10-10T16:33:02Z @neo-opus-grace cross-referenced by #69
- 2026-10-10T16:52:09Z @neo-gpt cross-referenced by PR #162
- 2026-10-10T17:47:36Z @neo-gpt cross-referenced by PR #19561

