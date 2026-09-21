---
id: 15
title: DevIndex has no version-update automation across 34 dependencies
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T16:32:15Z'
updatedAt: '2026-09-15T18:57:47Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/15'
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
closedAt: '2026-09-15T18:57:47Z'
---
# DevIndex has no version-update automation across 34 dependencies

> [!IMPORTANT]
> **Corrected by the author before implementation.** This ticket's Out of Scope claimed the config *"does not fix"* the skills staleness because *"with a caret, dependabot never proposes the bump, the range already permits the latest."* **That is wrong** — @neo-opus-grace measured the lockfiles: `neo-agent-brain` 0.1.3, `neo-agent-institution` 0.1.6, `devindex` 0.1.1. A caret does **not** float once a lockfile exists, so those repos are deterministically **stale**, not variable — and dependabot updates a lockfile that lags its range. So this config is expected to **surface** the skills bump as a standalone PR (the exclusion keeps it out of the bulk group), which makes it more useful than the ticket claimed, not less. The caret→exact question survives as a separate readability/policy matter, not as a "config is inert" one.

## Context

Surfaced by @tobiu while `neomjs/neo`'s skills pin moved to `0.1.7` (`neomjs/neo#18744` / `#18745`). Measured across the org on 2026-09-15: **`neomjs/neo` is the only repository with `.github/dependabot.yml`.** This repository, the Brain, the Institution and the Skills repo have none.

`neomjs/neo`'s config states why nothing propagates it:

> *"No materializer, installer or drift check accompanies this file: each repository commits its own."*

## The Problem

**33 dev dependencies, 1 runtime, 3 workflows — no automated version updates.** Security updates run without config; version updates do not. The three workflows pin action versions, and per `neomjs/neo`'s own comment, *"nothing else updates the pinned action versions."*

This repository also carries the org's **stalest declared** skills pin, `^0.1.1`, against a published `0.1.7`.

## The Architectural Reality

`neomjs/neo`'s config is the reference implementation, and two of its decisions are policy rather than taste:

- **Majors are included by not being excluded** — the absence of a `semver-major` ignore key is deliberate and documented.
- **Grouped, not per-dependency** — *"ungrouped daily updates open a PR per dependency and the queue stops being read."*

**Both of its exclusions transfer here**, for the reasons it records:

- `neo-agent-skills` (`^0.1.1`) — `postinstall` runs `neo-agent-skills-materialize`, so a release rewrites the workflows every agent in this checkout obeys; that move must be read in a standalone PR, not skimmed inside a bulk bump.
- `monaco-editor` (`0.50.0`, exact) — a bump requires addon/wrapper compatibility checks, so its diff has to be read.

`neo.mjs` is `^13.1.0` from the registry here, so unlike the Brain and Institution it is within dependabot-npm's reach and needs no carve-out.

## The Fix

Add `.github/dependabot.yml`: `npm` grouped with `exclude-patterns: ["monaco-editor", "neo-agent-skills"]`, plus `github-actions` grouped. Carry the majors-by-omission comment and both exclusion rationales.

## Decision Record impact

`none`. Adopting an existing, documented org pattern.

## Acceptance Criteria

- [ ] `.github/dependabot.yml` exists, valid `version: 2`, both ecosystems grouped.
- [ ] `monaco-editor` and `neo-agent-skills` are both excluded from the npm group, each with its rationale recorded in the file — both packages are present here, so neither pattern is dead config.
- [ ] No `semver-major` ignore key, and a comment recording that the omission is the policy.
- [ ] No carve-out for `neo.mjs` — it is a registry range here, and an exclusion would remove a dependency dependabot can actually track.

## Out of Scope

- **Moving `neo-agent-skills` from `^0.1.1` to an exact pin.** The caret resolves to `0.1.7` today, so the declared pin reads six releases behind while a fresh install is current — install date decides which review rules agents here obey. **the ticket originally mis-stated this — see the banner**: dependabot updates a lockfile lagging its range, so it is expected to propose this as a standalone PR. Measurement is on `neo-agent-skills#14`; the fix needs its own lane.
- Any version bump in this PR.

## Avoided Traps

**Do not read the stale-looking `^0.1.1` as the version in use.** A fresh install resolves `0.1.7`. The declared pin and the loaded corpus are different facts, and only the second governs what an agent reads — assert the installed file, never the pin.

**Do not add a `semver-major` ignore.** It looks prudent and silently reverses a documented decision.

**Do not copy `neomjs/neo`'s file verbatim** — its exclusions happen to transfer, but the reasoning must be re-checked per repository rather than inherited. Both were verified present here.

## Related

- `neo-agent-skills#14` — cross-repo governance epic; the org-wide measurement lives in its thread.
- `neomjs/neo#18744` / `#18745` — the Engine move that surfaced this.
- Sibling filings: `neo-agent-skills`, `neo-agent-brain`, `neo-agent-institution`.

**Sweeps.** Live latest-open: all 3 open issues here at 2026-09-15T16:2xZ (`#10`, `#9`, `#1`) plus an exact `dependabot` search across all states — zero results. A2A in-flight claim sweep, latest 30 messages all read-states: no claim on dependency automation in any repository. Memory Core rationale sweep on the problem's nouns: no prior decision. Own-assignment sweep: nothing of mine open here (`#13` closed retracted earlier today).

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `dependabot absent devindex version updates monaco skills exclusions caret resolves current`


## Timeline

- 2026-09-15T16:32:16Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T16:32:17Z @neo-opus-vega added the `enhancement` label
- 2026-09-15T16:32:17Z @neo-opus-vega added the `ai` label
- 2026-09-15T16:43:22Z @neo-opus-vega cross-referenced by PR #16
- 2026-09-15T18:57:47Z @tobiu referenced in commit `924cdff` - "Merge pull request #16 from neomjs/agent/15-dependabot

34 dependencies and three workflows get a version-update surface (#15)"
- 2026-09-15T18:57:47Z @tobiu closed this issue

