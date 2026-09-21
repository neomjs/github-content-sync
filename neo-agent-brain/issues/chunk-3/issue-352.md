---
id: 352
title: The Brain has no version-update automation across 30 dependencies and 10 workflows
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T16:31:37Z'
updatedAt: '2026-09-15T17:42:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/352'
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
closedAt: '2026-09-15T17:42:30Z'
---
# The Brain has no version-update automation across 30 dependencies and 10 workflows

> [!IMPORTANT]
> **Corrected by the author before implementation.** This ticket's Out of Scope claimed the config *"does not fix"* the skills staleness because *"with a caret, dependabot never proposes the bump, the range already permits the latest."* **That is wrong** — @neo-opus-grace measured the lockfiles: `neo-agent-brain` 0.1.3, `neo-agent-institution` 0.1.6, `devindex` 0.1.1. A caret does **not** float once a lockfile exists, so those repos are deterministically **stale**, not variable — and dependabot updates a lockfile that lags its range. So this config is expected to **surface** the skills bump as a standalone PR (the exclusion keeps it out of the bulk group), which makes it more useful than the ticket claimed, not less. The caret→exact question survives as a separate readability/policy matter, not as a "config is inert" one.

## Context

Surfaced by @tobiu while `neomjs/neo`'s skills pin moved to `0.1.7` (`neomjs/neo#18744` / `#18745`). Measured across the org on 2026-09-15: **`neomjs/neo` is the only repository with `.github/dependabot.yml`.** The Brain, Institution, DevIndex and the Skills repo have none.

`neomjs/neo`'s config states why nothing propagates it:

> *"No materializer, installer or drift check accompanies this file: each repository commits its own."*

## The Problem

This repository carries **24 runtime + 6 dev dependencies and 10 workflows**, and receives **no automated version updates**. Security updates run without config; version updates do not. The runtime set includes native and vector components — `better-sqlite3@12.11.1`, `chromadb@3.5.0`, `@chroma-core/default-embed@0.1.9`, `express@5.2.1` — all exact-pinned, which is correct for reproducibility and also means every move is manual and currently unprompted.

The 10 workflows pin action versions, and per `neomjs/neo`'s own comment, *"nothing else updates the pinned action versions."* A security surface like `#300` (sharp's libvips CVEs) is exactly the class that arrives as a version bump nobody is watching for.

## The Architectural Reality

`neomjs/neo`'s config is the reference implementation, and two of its decisions are policy rather than taste:

- **Majors are included by not being excluded** — the absence of a `semver-major` ignore key is deliberate and documented.
- **Grouped, not per-dependency** — *"ungrouped daily updates open a PR per dependency and the queue stops being read."*

**One exclusion transfers and one does not.** `neo-agent-skills` (`^0.1.3` here) belongs excluded for the reason `neomjs/neo` records verbatim — `postinstall` runs `neo-agent-skills-materialize`, so a release rewrites the workflows every agent in this checkout obeys, and that move must be read in a standalone PR rather than skimmed inside a bulk bump. `monaco-editor` is not a dependency here, so its exclusion would be dead config.

**`neo.mjs` is a tarball URL** pinned to a commit (`https://github.com/neomjs/neo/archive/17b59aad…tar.gz`). Dependabot's npm ecosystem cannot bump that form, so it stays manual regardless of this change — worth knowing, not worth solving here.

## The Fix

Add `.github/dependabot.yml`: `npm` grouped with `exclude-patterns: ["neo-agent-skills"]`, plus `github-actions` grouped. Carry the majors-by-omission comment and the exclusion rationale.

## Decision Record impact

`none`. Adopting an existing, documented org pattern.

## Acceptance Criteria

- [ ] `.github/dependabot.yml` exists, valid `version: 2`, both ecosystems grouped.
- [ ] `neo-agent-skills` is excluded from the npm group, with the symlink/materialize rationale recorded in the file.
- [ ] `monaco-editor` is **not** in `exclude-patterns` — it is not a dependency here, and a pattern for an absent package reads as a governing rule while governing nothing.
- [ ] No `semver-major` ignore key, and a comment recording that the omission is the policy.
- [ ] A comment records that `neo.mjs`'s tarball-URL form is outside dependabot-npm's reach, so no reader mistakes the config for full coverage.

## Out of Scope

- **Moving `neo-agent-skills` from `^0.1.3` to an exact pin.** The caret resolves to `0.1.7` today, so install date decides which review rules agents here obey — and **the ticket originally mis-stated this — see the banner**: dependabot updates a lockfile lagging its range, so it is expected to propose this as a standalone PR. Measurement is on `neo-agent-skills#14`; the fix needs its own lane.
- `#300` / sharp — a specific vulnerable dependency, owned there.
- Adopting the shared PR baseline. This repository calls no baseline workflow at all, which is `neomjs/neo#17783` / `neo-agent-skills#14` territory.
- Any version bump in this PR.

## Avoided Traps

**Do not copy `neomjs/neo`'s file verbatim** — its `monaco-editor` exclusion is meaningless here.

**Do not add a `semver-major` ignore.** It looks prudent and silently reverses a documented decision.

**Do not read the config as coverage.** The `neo.mjs` tarball and the `neo-agent-skills` caret both sit outside what it can act on.

## Related

- `neo-agent-skills#14` — cross-repo governance epic; the org-wide measurement lives in its thread.
- `neomjs/neo#18744` / `#18745` — the Engine move that surfaced this.
- Sibling filings: `neo-agent-skills`, `neo-agent-institution`, `devindex`.

**Sweeps.** Live latest-open: newest 8 open issues here at 2026-09-15T16:2xZ (#349 … #300) plus an exact `dependabot` search across all states — `#300` (sharp CVEs) and `#321` (closed, defect ledger) only; neither is this. A2A in-flight claim sweep, latest 30 messages all read-states: no claim on dependency automation. Memory Core rationale sweep on the problem's nouns: no prior decision. Own-assignment sweep: `#342` is mine here and unrelated.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `dependabot absent brain version updates github-actions skills exclusion tarball unbumpable`


## Timeline

- 2026-09-15T16:31:37Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T16:31:38Z @neo-opus-vega added the `enhancement` label
- 2026-09-15T16:31:38Z @neo-opus-vega added the `ai` label
- 2026-09-15T16:42:33Z @neo-opus-vega cross-referenced by PR #355
- 2026-09-15T17:42:30Z @tobiu referenced in commit `1930e2b` - "Merge pull request #355 from neomjs/agent/352-dependabot

30 dependencies and 10 workflows get a version-update surface (#352)"
- 2026-09-15T17:42:30Z @tobiu closed this issue
- 2026-09-15T18:03:07Z @neo-opus-vega cross-referenced by PR #357
- 2026-09-15T18:16:31Z @neo-opus-vega cross-referenced by PR #359

