---
id: 218
title: Brain unit setup names nonexistent install artifacts
state: CLOSED
labels:
  - bug
  - developer-experience
  - ai
  - testing
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-28T23:06:49Z'
updatedAt: '2026-08-28T23:42:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/218'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 194
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-28T23:42:45Z'
---
# Brain unit setup names nonexistent install artifacts

## Context

Brain's unit configuration correctly fails closed in CI when its dependency tier is absent and skips that tier loudly for an incomplete local install. Both messages then prescribe artifacts that do not exist.

Measured on current `dev@1e50a625`:

- `package.json` has no `install-brain` script;
- `package.brain.json` is not tracked;
- Brain already declares `better-sqlite3`, `chromadb`, and `@chroma-core/default-embed` in its root manifest;
- a normal root install is the supported dependency path.

This is split from #201 because correcting two developer-facing instructions is one reviewable PR; binding the full unit corpus is blocked by 91 current failures across 27 files.

## The Problem

`assertBrainTierForEnvironment()` tells CI users to run `npm run install-brain`. The local skip line points to `package.brain.json` and repeats the same command. Neither remedy is executable, so the guard detects the right failure and strands the developer at a dead end.

The stale wording was inherited from the Engine's former optional Brain tier. Inside the Brain repository, the root manifest is already the dependency authority.

## The Architectural Reality

`test/playwright/playwright.config.unit.mjs` owns both the dependency-tier probe and the human-facing recovery text. The gate's fail-closed CI behavior is correct and must not change. This leaf updates guidance only; #201 remains the terminal full-suite CI binding.

## The Fix

Replace the optional-tier history and nonexistent artifact names with one current setup instruction derived from the root manifest:

- normal local recovery: run `npm ci`;
- when reproducing CI's intentional `--ignore-scripts` install, rebuild `better-sqlite3` before the suite.

Keep a single message constant so the local skip and CI error cannot drift. Add a focused unit witness that rejects the retired names and verifies every named setup artifact/command exists in the current repository contract.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| `assertBrainTierForEnvironment()` CI error | Brain root manifest + `brain-unit.yml` install sequence | Fails closed and names the current install/rebuild path | Missing tier still throws before collection | JSDoc in config | focused unit witness |
| Local missing-tier info line | Brain root manifest | Skips loudly and tells the developer how to restore root dependencies | Never claims the partial run is full-unit green | inline config comment | focused unit witness |
| Brain-tier project selection | existing `buildProjects()` contract | Unchanged | CI remains fail-closed; local incomplete install remains loud | none | existing list control + focused spec |

## Decision Record impact

None. This restores current developer guidance without changing ADR 0040 topology or #212 test architecture.

## Acceptance Criteria

- [ ] `playwright.config.unit.mjs` contains no `install-brain` or `package.brain.json` reference.
- [ ] CI missing-tier failure still occurs before collection and names an executable current recovery path.
- [ ] Local missing-tier output remains loud and names the same root dependency authority.
- [ ] A focused unit test proves the retired names stay absent and the current setup guidance matches tracked manifest/workflow surfaces.
- [ ] Brain-tier detection, project selection, worker count, retries, and full-suite scope are unchanged.
- [ ] The current three-spec CI smoke and #201's 91-failure burndown are not modified here.

## Out of Scope

Running the full retained suite in CI, repairing its current failures, adding an optional dependency manifest, or changing dependency topology.

## Avoided Traps

- Restoring `package.brain.json` inside the Brain repository.
- Adding an `install-brain` alias for a tier that is already the root package.
- Weakening the CI throw to make an incomplete install green.

## Related

Parent: #194. Related: #201 and #89.

Origin Session ID: `45cace1c-f22b-4e00-bdc0-610f6e5f4b7e`.

Retrieval Hint: `Brain unit tier setup guidance install-brain package.brain.json root npm ci`.

Live latest-open and A2A claim sweeps: checked immediately before creation on 2026-08-29; no equivalent ticket or competing claim found.

## Timeline

- 2026-08-28T23:06:51Z @neo-gpt-emmy added the `bug` label
- 2026-08-28T23:06:51Z @neo-gpt-emmy added the `developer-experience` label
- 2026-08-28T23:06:51Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T23:06:52Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T23:06:52Z @neo-gpt-emmy added the `build` label
- 2026-08-28T23:06:52Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T23:07:01Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-28T23:12:38Z @neo-gpt-emmy cross-referenced by PR #219
- 2026-08-28T23:32:45Z @neo-opus-vega cross-referenced by PR #220
- 2026-08-28T23:40:51Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-28T23:42:45Z @tobiu referenced in commit `4620628` - "Merge pull request #219 from neomjs/codex/218-unit-setup-guidance

fix(test): correct Brain unit setup guidance (#218)"
- 2026-08-28T23:42:45Z @tobiu closed this issue

