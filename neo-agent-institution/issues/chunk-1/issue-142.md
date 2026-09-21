---
id: 142
title: The Institution has no version-update automation across 30 dependencies
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T16:31:59Z'
updatedAt: '2026-09-15T17:02:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/142'
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
closedAt: '2026-09-15T17:02:14Z'
---
# The Institution has no version-update automation across 30 dependencies

> [!IMPORTANT]
> **Corrected by the author before implementation.** This ticket's Out of Scope claimed the config *"does not fix"* the skills staleness because *"with a caret, dependabot never proposes the bump, the range already permits the latest."* **That is wrong** — @neo-opus-grace measured the lockfiles: `neo-agent-brain` 0.1.3, `neo-agent-institution` 0.1.6, `devindex` 0.1.1. A caret does **not** float once a lockfile exists, so those repos are deterministically **stale**, not variable — and dependabot updates a lockfile that lags its range. So this config is expected to **surface** the skills bump as a standalone PR (the exclusion keeps it out of the bulk group), which makes it more useful than the ticket claimed, not less. The caret→exact question survives as a separate readability/policy matter, not as a "config is inert" one.

## Context

Surfaced by @tobiu while `neomjs/neo`'s skills pin moved to `0.1.7` (`neomjs/neo#18744` / `#18745`). Measured across the org on 2026-09-15: **`neomjs/neo` is the only repository with `.github/dependabot.yml`.** This repository, the Brain, DevIndex and the Skills repo have none.

`neomjs/neo`'s config states why nothing propagates it:

> *"No materializer, installer or drift check accompanies this file: each repository commits its own."*

## The Problem

**28 dev dependencies, 2 runtime, 2 workflows — no automated version updates.** Security updates run without config; version updates do not. `#124` (vulnerable `sharp` inherited through the embedding dependency) is exactly the class that arrives as a version bump nobody is prompted to look for.

The two workflows pin action versions, and per `neomjs/neo`'s own comment, *"nothing else updates the pinned action versions."*

## The Architectural Reality

`neomjs/neo`'s config is the reference implementation, and two of its decisions are policy rather than taste:

- **Majors are included by not being excluded** — the absence of a `semver-major` ignore key is deliberate and documented.
- **Grouped, not per-dependency** — *"ungrouped daily updates open a PR per dependency and the queue stops being read."*

**Both of its exclusions transfer here**, for the reasons it records:

- `neo-agent-skills` (`^0.1.6`) — `postinstall` runs `neo-agent-skills-materialize`, so a release rewrites the workflows every agent in this checkout obeys; that move must be read in a standalone PR, not skimmed inside a bulk bump.
- `monaco-editor` (`0.50.0`, exact) — a bump requires addon/wrapper compatibility checks, so its diff has to be read.

**Two dependencies are `github:` refs pinned to commits** — `neo-agent-brain#bd541715…` and `neo.mjs#337c9fd5…`. Dependabot's npm ecosystem cannot bump that form, so those stay manual regardless of this change. Worth recording so the config is not mistaken for full coverage.

## The Fix

Add `.github/dependabot.yml`: `npm` grouped with `exclude-patterns: ["monaco-editor", "neo-agent-skills"]`, plus `github-actions` grouped. Carry the majors-by-omission comment and both exclusion rationales.

## Decision Record impact

`none`. Adopting an existing, documented org pattern.

## Acceptance Criteria

- [ ] `.github/dependabot.yml` exists, valid `version: 2`, both ecosystems grouped.
- [ ] `monaco-editor` and `neo-agent-skills` are both excluded from the npm group, each with its rationale recorded in the file — both packages are present here, so neither pattern is dead config.
- [ ] No `semver-major` ignore key, and a comment recording that the omission is the policy.
- [ ] A comment records that the two `github:` commit-pinned dependencies are outside dependabot-npm's reach.

## Out of Scope

- **Moving `neo-agent-skills` from `^0.1.6` to an exact pin.** The caret resolves to `0.1.7` today, so install date decides which review rules agents here obey — and **this config does not fix it**: dependabot updates a lockfile lagging its range, so it is expected to propose this as a standalone PR. Measurement is on `neo-agent-skills#14`; the fix needs its own lane.
- `#124` / sharp — a specific vulnerable dependency, owned there.
- Any version bump in this PR.

## Avoided Traps

**Do not add a `semver-major` ignore.** It looks prudent and silently reverses a documented decision.

**Do not read the config as coverage.** The two `github:` refs and the `neo-agent-skills` caret both sit outside what it can act on — a reader who assumes otherwise stops checking them by hand.

## Related

- `neo-agent-skills#14` — cross-repo governance epic; the org-wide measurement lives in its thread.
- `neomjs/neo#18744` / `#18745` — the Engine move that surfaced this.
- `#138`/`#139` and `#140`/`#141` — @neo-opus-grace's skills-pin lanes here.
- Sibling filings: `neo-agent-skills`, `neo-agent-brain`, `devindex`.

**Sweeps.** Live latest-open: newest 8 open issues here at 2026-09-15T16:2xZ (#129 … #19) plus an exact `dependabot` search across all states — only `#124` (sharp) returned; not this. A2A in-flight claim sweep, latest 30 messages all read-states: @neo-opus-grace holds the skills-**pin** lanes here (`#138`–`#141`), which this deliberately does not touch; no claim on dependency automation. Memory Core rationale sweep on the problem's nouns: no prior decision. Own-assignment sweep: nothing of mine here.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `dependabot absent institution version updates monaco skills exclusions github refs unbumpable`


## Timeline

- 2026-09-15T16:31:59Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T16:32:00Z @neo-opus-vega added the `enhancement` label
- 2026-09-15T16:32:01Z @neo-opus-vega added the `ai` label
- 2026-09-15T16:42:56Z @neo-opus-vega cross-referenced by PR #143
- 2026-09-15T17:02:14Z @tobiu referenced in commit `f486ed3` - "Merge pull request #143 from neomjs/agent/142-dependabot

30 dependencies and two workflows get a version-update surface (#142)"
- 2026-09-15T17:02:14Z @tobiu closed this issue
- 2026-09-18T10:25:56Z @neo-fable-clio cross-referenced by #149

