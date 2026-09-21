---
id: 88
title: 'Nothing retires a stale npm override, and Dependabot ignores `overrides`'
state: CLOSED
labels:
  - enhancement
  - dependencies
  - ai
  - build
assignees:
  - neo-opus-ada
createdAt: '2026-09-18T09:47:54Z'
updatedAt: '2026-09-18T10:49:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/88'
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
closedAt: '2026-09-18T10:49:09Z'
---
# Nothing retires a stale npm override, and Dependabot ignores `overrides`

## Context

The operator asked (2026-09-18) whether Dependabot will remove our npm `overrides` once the packages that needed them catch up. It will not. dependabot-core#5590 ("handle the `overrides` field", open since 2022) and dependabot-core#14736 (feature request, open) show that Dependabot neither bumps, adds nor removes `overrides` entries.

Overrides today: `neomjs/neo` has three (`gray-matter > js-yaml ^3.15.2`, `lodash-es ^4.18.0`, `monaco-editor > dompurify ^3.4.13`), and `neomjs/neo-agent-brain` has one (`gray-matter > js-yaml ^3.15.2`). `neo-agent-institution` and `devindex` have none.

## The Problem

An override is a floor. It keeps a dependent whose declared range admits a vulnerable version from resolving below the patched one. The lockfile carries the remediation itself, as @neo-opus-grace established on neo#18601 / brain#346. Once every dependent's declared range starts at or above the floor, the override does nothing, and nothing notices. Two harms follow:

1. **Dead config accretes.** Each override's retirement condition lives only in its ticket (neo#18600, neo#18354, neo#18812).
2. **A stale override can force a downgrade.** If `gray-matter` ships a release declaring `js-yaml ^4`, our `^3.15.2` override silently forces it back onto js-yaml 3.x, because npm overrides beat declared ranges.

## The Architectural Reality

- Everything the check needs is already in the consumer's `package.json` (the rules) and `package-lock.json` (every installed package's declared `dependencies` / `optionalDependencies` / `peerDependencies`). No network is needed.
- The distribution pattern already exists: `bin` checks (`check-pr-body`, `check-substrate-size`, `check-workflow-concurrency`, …) plus jobs in `.github/workflows/reusable-pr-baseline.yml`, which each consumer calls.
- The package already ships a runtime dependency (`acorn`), so npm's own `semver` can be one too, which keeps the range math identical to npm's.

**Prototype, measured on neo `7b19aa730c`** (it borrows neo's `semver`):

| rule | verdict | dependents still below the floor |
|---|---|---|
| `gray-matter > js-yaml ^3.15.2` | needed | `gray-matter` declares `^3.13.1` |
| `lodash-es ^4.18.0` | needed | `chevrotain`, `@chevrotain/gast` and `@chevrotain/cst-dts-gen` pin `4.17.23`; `dagre-d3-es` declares `^4.17.21` |
| `monaco-editor > dompurify ^3.4.13` | needed | `monaco-editor` pins `3.4.8` |

Controls, on a copy of the lock with `gray-matter`'s declared js-yaml range rewritten: `^4.1.0` → **FIGHTING**, `^3.15.2` → **REDUNDANT**. The classifier separates all three states.

## The Fix

Add a `bin` check, `neo-agent-skills-npm-overrides`. For every rule in `overrides`, top-level or nested under a parent, it collects each dependent edge's declared range for the target from the lock; for a nested rule, only dependents inside the named parent's resolved subtree count, hoisted ones included, because that is where npm applies it. It classifies the rule as:

- **needed:** some declared range's minimum is below the override's floor. Print those edges; they are the retirement condition, recorded mechanically.
- **REDUNDANT:** every declared range's minimum is at or above the floor. Fail, and name the entry to delete.
- **FIGHTING:** some declared range does not intersect the override and starts above it, so the override forces a downgrade. Fail.

An override that forces a version *above* an exact pin (the monaco case) is the deliberate security direction, so it stays `needed`.

Wire it as a `reusable-pr-baseline.yml` job on every pull request. The verdict is a pure function of `package.json` and `package-lock.json`, so it can only change in a PR that changes one of them, such as the Dependabot PR that moves a declared range. Unrelated PRs therefore never go red from upstream drift, and no path gate is needed. With no `overrides` present, the job exits 0, so all four consumers can call it uniformly. Then release, and bump the pin in `neo` and `neo-agent-brain`.

## Acceptance Criteria

- [ ] The bin classifies each rule as needed, REDUNDANT or FIGHTING from `package.json` plus `package-lock.json` alone. Unit arms cover all three states and nested versus top-level rules, and the two controls above are among them.
- [ ] It exits 0 when there are no `overrides`, and exits non-zero on any REDUNDANT or FIGHTING rule, naming the rule and the edges.
- [ ] The baseline job runs on every pull request. It installs the guard from `runner.temp` at the pinned release and runs it bare in the caller workspace, and the workflow-contract suite asserts each of these.
- [ ] Range math is npm's own `semver`, declared as a runtime dependency.
- [ ] Run against neo and brain at their current `dev`, it reports all four rules as `needed`.

*(Corrected 2026-09-18 during implementation: the job runs on every PR because the verdict reads only the two package files, which makes a path gate unnecessary; and the package was never dependency-free.)*

## Out of Scope

- pnpm and yarn resolutions: every org repo uses npm.
- Auto-removing an override: the check names the line, and a human or agent deletes it in the same PR.
- Advisory lookups: whether the floor is still the patched version is the ticket's and the advisory's business, not this check's.

## Avoided Traps

- **Re-resolving without the override (`npm install --package-lock-only`):** it needs network and a registry. It also passes vacuously, because npm keeps an already-locked version that satisfies the range, so the floor looks unneeded while a dependent still admits the vulnerable version.
- **A scheduled run, or any network re-resolution:** either lets an upstream release red a PR that changed nothing.
- **Reading `npm ls` `overridden` markers:** they mark that a rule applied to an edge, not whether the rule changed the outcome.

## Related

neo#18833 (the same session's lock discovery) · neo#18600 · neo#18354 · neo#18812 · dependabot-core#5590 · dependabot-core#14736

Live latest-open sweep: checked the latest 20 open issues in this repo at 2026-09-18T09:55Z, and none covers overrides. An all-states search for "overrides" hits only #51, which is unrelated. Memory Core sweep: prior art on override semantics, and no prior decision on a check. Own-assigned sweep: #63, #65, #66 and #76 are on other surfaces.

Retrieval Hint: "stale npm override retirement check dependabot ignores overrides"

Origin Session ID: 6ed6621f-e2e4-4b9d-9a95-744679cb3c59


## Timeline

- 2026-09-18T10:07:45Z @neo-opus-ada cross-referenced by PR #89
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149

