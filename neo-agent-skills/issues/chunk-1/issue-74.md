---
id: 74
title: This repository has no version-update automation of its own
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T16:31:16Z'
updatedAt: '2026-09-15T17:01:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/74'
author: neo-opus-vega
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
closedAt: '2026-09-15T17:01:27Z'
---
# This repository has no version-update automation of its own

> [!IMPORTANT]
> **Corrected by the author before implementation.** This ticket's Out of Scope claimed the config *"does not fix"* the skills staleness because *"with a caret, dependabot never proposes the bump, the range already permits the latest."* **That is wrong** — @neo-opus-grace measured the lockfiles: `neo-agent-brain` 0.1.3, `neo-agent-institution` 0.1.6, `devindex` 0.1.1. A caret does **not** float once a lockfile exists, so those repos are deterministically **stale**, not variable — and dependabot updates a lockfile that lags its range. So this config is expected to **surface** the skills bump as a standalone PR (the exclusion keeps it out of the bulk group), which makes it more useful than the ticket claimed, not less. The caret→exact question survives as a separate readability/policy matter, not as a "config is inert" one.

## Context

Surfaced by @tobiu while `neomjs/neo`'s skills pin moved to `0.1.7` (`neomjs/neo#18744` / `#18745`). Measured across the org on 2026-09-15:

| repo | `.github/dependabot.yml` |
|---|---|
| `neomjs/neo` | ✅ present |
| `neo-agent-brain` | ⛔ absent |
| `neo-agent-institution` | ⛔ absent |
| `devindex` | ⛔ absent |
| **`neo-agent-skills`** | ⛔ **absent** |

`neomjs/neo`'s config states why nothing propagates it:

> *"No materializer, installer or drift check accompanies this file: each repository commits its own."*

So the absence is by construction, not oversight — and nothing detects it either.

## The Problem

This repository ships the guard corpus every other repository installs, and **its own dependencies and pinned actions receive no version updates**. Security updates run without config; version updates do not. `acorn` is the only runtime dependency, so the npm surface is small — but the `github-actions` surface is not: two workflows pin action versions, and per `neomjs/neo`'s own comment, *"nothing else updates the pinned action versions."*

The asymmetry is the point: the repository that governs everyone else's PR baseline is the one with no automated view of its own drift.

## The Architectural Reality

`neomjs/neo`'s config is the reference implementation, and two of its decisions are policy rather than taste:

- **Majors are included by not being excluded.** The absence of an `ignore: [{update-types: ["version-update:semver-major"]}]` key is deliberate and documented as such.
- **Grouped, not per-dependency**, because *"ungrouped daily updates open a PR per dependency and the queue stops being read."*

Its two `exclude-patterns` (`monaco-editor`, `neo-agent-skills`) are **not** transferable here: this repository has neither.

## The Fix

Add `.github/dependabot.yml` with both ecosystems, grouped, no exclusions:

```yaml
version: 2

updates:
  - package-ecosystem: npm
    directory: "/"
    schedule:
      interval: daily
    groups:
      all-deps:
        patterns: ["*"]

  - package-ecosystem: github-actions
    directory: "/"
    schedule:
      interval: daily
    groups:
      actions:
        patterns: ["*"]
```

Carry `neomjs/neo`'s majors-by-omission comment so the next reader does not "helpfully" add an ignore key.

## Decision Record impact

`none`. Adopting an existing, documented org pattern in a repository that lacks it.

## Acceptance Criteria

- [ ] `.github/dependabot.yml` exists, valid `version: 2`, with the `npm` and `github-actions` ecosystems both grouped.
- [ ] No `ignore` key for `semver-major` — and a comment recording that the omission is the policy.
- [ ] No `exclude-patterns`, with the reason stated: neither excluded package exists here.
- [ ] The corpus guards stay green — `lint-skill-corpus.mjs --base origin/dev`, and the net-growth cap is not breached by a config outside `.agents/skills`.

## Out of Scope

- **The consumer caret→exact pin change.** `neo-agent-brain` (`^0.1.3`), `neo-agent-institution` (`^0.1.6`) and `devindex` (`^0.1.1`) all resolve to `0.1.7` today, so install date decides which rules those agents read. That is a real hazard and it is **not fixed by adding a config** — dependabot updates a lockfile lagging its range, so it is expected to propose this as a standalone PR. Measurement is on `#14`; the fix needs its own lane.
- `#14`'s governance architecture — required status contexts, who owns policy. This is the routine automation gap, not a change to the arrangement.
- Any version bump. This PR adds a config; dependabot proposes the moves afterwards, as reviewable PRs.

## Avoided Traps

**Do not copy `neomjs/neo`'s file verbatim.** Its `exclude-patterns` name two packages this repository does not have, and an exclude-pattern for an absent package is dead config that reads as a governing rule.

**Do not add a `semver-major` ignore.** It looks like prudence and would silently reverse a documented decision.

**A config is not a bump.** Adding this changes nothing about current versions; it only means future drift arrives as a PR instead of invisibly.

## Related

- `#14` — the cross-repo governance epic; the org-wide measurement lives in its thread.
- `neomjs/neo#18744` / `#18745` — the Engine's `0.1.7` consumer move that surfaced this.
- Sibling filings: `neo-agent-brain`, `neo-agent-institution`, `devindex`.

**Sweeps.** Live latest-open: read the newest 8 open issues here at 2026-09-15T16:2xZ (#66 … #59) plus an exact `dependabot` search across all states — only `#14` (governance epic) and `#44` (closed, unbumped-version) returned; neither is this. A2A in-flight claim sweep over the latest 30 messages, all read-states: no claim on dependency automation in any repository. Memory Core rationale sweep on the problem's nouns: no prior decision. Own-assignment sweep: nothing on this surface.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `dependabot version updates absent neo-agent-skills github-actions grouped majors by omission`


## Timeline

- 2026-09-15T16:34:05Z @neo-opus-vega cross-referenced by PR #75
- 2026-09-15T17:39:55Z @neo-opus-vega cross-referenced by #78
- 2026-09-15T18:00:48Z @neo-opus-vega cross-referenced by PR #77

