---
id: 94
title: Bump neo.mjs from ^13.2.0 to ^13.3.1
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-gemini-julia
createdAt: '2026-10-23T23:45:22Z'
updatedAt: '2026-10-24T00:22:15Z'
githubUrl: 'https://github.com/neomjs/devindex/issues/94'
author: neo-gemini-julia
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
closedAt: '2026-10-24T00:22:14Z'
---
# Bump neo.mjs from ^13.2.0 to ^13.3.1

## Context

Engine `v13.3.1` is published on npm (`neo.mjs@13.3.1`, gitHead `dcae85851f`, atomic release commit `24d865eb06`). As part of the `neomjs/neo#20022` (AC-8) post-release consumer rollout handed off by `@neo-opus-vega` (`MESSAGE:e50b149c-b2d0-4102-813b-4be20adf5cbe`), `neomjs/devindex` needs to advance its `neo.mjs` dependency from `^13.2.0` to `^13.3.1`.

## The Problem

`neomjs/devindex` currently pins `"neo.mjs": "^13.2.0"` in `package.json` and resolves `13.2.0` in `package-lock.json` (having skipped `13.3.0`). Until bumped and locked to `13.3.1`, DevIndex builds and Pages deployments (`pages.yml`) continue bundling `neo.mjs@13.2.0` rather than the current `13.3.1` engine release.

## The Architectural Reality

-Both `13.3.0` (`Upgrading from 13.2`) and `13.3.1` (`Upgrading from 13.3.0`) are drop-in replacements for `neomjs/devindex`: package entry points (`main`, `exports`, `bin`, `engines`, `type`) are unchanged, and `apps/`, `services/`, and `buildScripts/` in `neomjs/devindex` have zero consumers of the changed edge surfaces (`mountedPromise`, `PortDisconnectedError`, `SharedCanvas`, `afterSetMounted`, `afterVnodeSync`, `Rectangle#intersects`).
- `ci.yml` runs `npm ci`, `npm run check-data-tracking`, `npx --no-install neo-agent-skills-materialize --check`, and `npx playwright test -c test/playwright/playwright.config.unit.mjs --project=unit --fail-on-flaky-tests`.
- `pages.yml` builds the application via `npm run build-all -- -l no -p no`.

## The Fix

1. Update `package.json` dependency `"neo.mjs"` from `"^13.2.0"` to `"^13.3.1"`.
2. Regenerate `package-lock.json` against npm so `node_modules/neo.mjs` resolves to `13.3.1`.
3. Verify `npm ci`, `npm run build-all -- -l no -p no`, `npm run check-data-tracking`, `npx --no-install neo-agent-skills-materialize --check`, and `npx playwright test -c test/playwright/playwright.config.unit.mjs --project=unit --fail-on-flaky-tests`.

## Decision Record impact

`none` — routine post-release consumer dependency upgrade.

## Acceptance Criteria

- [ ] AC-1 `package.json` declares `"neo.mjs": "^13.3.1"` and `package-lock.json` resolves `node_modules/neo.mjs` to `13.3.1`.
- [ ] AC-2 `npm run build-all -- -l no -p no` exits `0` against `neo.mjs@13.3.1` without errors.
- [ ] AC-3 `npm run check-data-tracking`, `npx --no-install neo-agent-skills-materialize --check`, and `npx playwright test -c test/playwright/playwright.config.unit.mjs --project=unit --fail-on-flaky-tests` pass (`47/47` unit tests).
- [ ] AC-4 Receipt posted on `neomjs/neo#20022` (AC-8).

## Out of Scope

- Enabling `dist/esm` and dev-mode mount entries on Pages (`#61`).
- Data-sync cursor hosting (`#9`).

## Related

- `neomjs/neo#20022` — Cut Engine v13.3.1 through the release PR (AC-8 consumer rollout)
- `#61` — Every DevIndex environment is live under `/devindex/` and linked from the Portal
- `#64` — DevIndex cannot build on `neo.mjs` 13.2: the Data Factory sits under `apps/`

Live latest-open sweep: checked all open issues on `neomjs/devindex` (`#61`, `#9`) at 2026-10-23T23:44:47Z; no equivalent found.
A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-23T23:44:53Z; handed off to `@neo-gemini-julia` by `@neo-opus-vega` in `MESSAGE:e50b149c-b2d0-4102-813b-4be20adf5cbe`, no competing claim.
Memory Core rationale sweep: `query_raw_memories` for `"devindex upgrade bump neo.mjs 13.3.1 13.2.0"` (5 results); confirmed consumer upgrade pattern.
Own-assignment sweep: 0 open issues assigned to `@neo-gemini-julia` on `neomjs/devindex`.
Structure map: N/A (consumer `package.json` / `package-lock.json` dependency bump only).

Origin Session ID: 7bbb249d-903b-4ad7-b69c-658b286ce4a0
Retrieval Hint: "devindex neo.mjs 13.3.1 post-release consumer bump #20022"


## Timeline

- 2026-10-23T23:45:22Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-23T23:45:24Z @neo-gemini-julia added the `enhancement` label
- 2026-10-23T23:45:24Z @neo-gemini-julia added the `ai` label
- 2026-10-23T23:45:24Z @neo-gemini-julia added the `dependencies` label
- 2026-10-23T23:49:29Z @neo-gemini-julia cross-referenced by PR #95
- 2026-10-23T23:49:49Z @neo-gemini-julia cross-referenced by #20022
- 2026-10-24T00:22:15Z @tobiu referenced in commit `da91e19` - "chore(deps): bump neo.mjs from ^13.2.0 to ^13.3.1 (#94) (#95)"
- 2026-10-24T00:22:15Z @tobiu closed this issue

