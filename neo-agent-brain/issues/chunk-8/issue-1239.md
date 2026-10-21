---
id: 1239
title: Add CONTRIBUTING.md for human and autonomous-agent contributors
state: OPEN
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-21T14:15:40Z'
updatedAt: '2026-10-21T14:15:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1239'
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
---
# Add CONTRIBUTING.md for human and autonomous-agent contributors

## Context

Following the external-agent contribution on `neomjs/neo#19952` / `neomjs/neo#19959` (`@Jah-yee`), `@tobiu` noted that while `neomjs/neo` has a root [`CONTRIBUTING.md`](https://github.com/neomjs/neo/blob/dev/CONTRIBUTING.md) covering local setup, test tiers, issue claiming (`good first issue` / `help wanted`), and autonomous-agent onboarding (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo <repo> --audience contributor`), `neomjs/neo-agent-brain` does not yet have a `CONTRIBUTING.md`.

## The Problem

1. External human and autonomous-agent contributors arriving at `neomjs/neo-agent-brain` have no root `CONTRIBUTING.md` explaining:
   - the difference between operating an agent team through [`neomjs/neo-agent-institution`](https://github.com/neomjs/neo-agent-institution) vs changing the Brain / Agent OS substrate itself;
   - how to clone and run the isolated unit suite (`npm install` + `npm run test-unit`, which requires only Node.js 24+ and needs no Docker, database server, or API keys) vs integration suites (`npm run test-integration-unified`, `npm run test-integration-parity`) and pre-PR lints (`npm run agent-preflight`);
   - why `npm run test-*` must be used instead of bare `npx playwright test` (the Playwright configs live under `test/playwright/`);
   - what `npm install` (`prepare`) does automatically (`ai/scripts/setup/initServerConfigs.mjs` local gitignored config overlays + `neo-agent-skills-materialize`);
   - how to pick up `good first issue` / `help wanted` issues, claim before starting, target `dev` (not `main`), and run `npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-brain --audience contributor` for contributor-scoped agent instructions.
2. In `README.md`, the `PRs Welcome` badge (`:13`) and `## Contributing` section (`:288-292`) do not yet link to `./CONTRIBUTING.md`.

## The Architectural Reality

- `package.json` defines `"prepare": "node ./ai/scripts/setup/prepare.mjs"`, `"test-unit": "playwright test -c test/playwright/playwright.config.unit.mjs"`, `"test-integration-unified": "playwright test -c test/playwright/playwright.config.integration.mjs"`, `"test-integration-parity": "playwright test -c test/playwright/playwright.config.integration-parity.mjs"`, and `"agent-preflight": "node ./ai/scripts/agent-preflight.mjs"`.
- `neo-agent-skills` (`scripts/generate-agents-md.mjs`) already supports `--repo neo-agent-brain --audience contributor`.
- Every relative link inside `CONTRIBUTING.md` must resolve to an actual file in `neomjs/neo-agent-brain` (or use an explicit canonical URL when referencing sibling repositories like `neomjs/neo` or `neomjs/neo-agent-institution`).

## The Fix

1. Author a root `CONTRIBUTING.md` in `neomjs/neo-agent-brain` modeled on `neomjs/neo/CONTRIBUTING.md` and tailored to the Brain's exact setup, scripts, package boundaries (`src/**`, `ai/**`, `cloud/**`, `deploy/**`, `test/**`), issue-claiming workflow (`good first issue` / `help wanted`, `dev` target branch), and AI agent contributor section (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-brain --audience contributor`).
2. Update `README.md` (`PRs Welcome` badge and `## Contributing` section) to link to `./CONTRIBUTING.md`.

## Decision Record impact

`none` — documentation and contributor-onboarding alignment with `neomjs/neo/CONTRIBUTING.md` and `ADR 0040` package boundaries.

## Acceptance Criteria

- [ ] AC-1: Root `CONTRIBUTING.md` exists in `neomjs/neo-agent-brain` covering starting points (running an institution via `neo-agent-institution` vs changing the Brain), zero-credential clone + `npm install` + `npm run test-unit` setup, why bare `npx playwright test` fails without `-c`, what `npm install` (`prepare`) materializes, issue claiming (`good first issue` / `help wanted`, `dev` target branch), and AI agent contributor instructions (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-brain --audience contributor`).
- [ ] AC-2: Every relative link in `CONTRIBUTING.md` and `README.md` resolves to an existing path in `neomjs/neo-agent-brain` (verified mechanically).
- [ ] AC-3: `README.md` links the `PRs Welcome` badge and `## Contributing` section to `./CONTRIBUTING.md`.

## Out of Scope

- Modifying `neo-agent-skills` generator sections (tracked separately if needed).
- Adding `CONTRIBUTING.md` to `neomjs/neo-agent-institution` (tracked in its own `neomjs/neo-agent-institution` ticket).

## Related

- `neomjs/neo/CONTRIBUTING.md`
- `neomjs/neo#19952`, `neomjs/neo#19959`

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at 2026-10-21T14:13Z; no equivalent found.
MC sweep: `neo-agent-brain neo-agent-institution CONTRIBUTING.md outside contributor setup loop 0101-orientation-contributor`, 10 results, no prior decision found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-brain` after `#1238` merge, none overlapping.
Structure map: `npm run ai:structure-map -- --files --loc` inspected (`ai/scripts/setup/prepare.mjs`, root `README.md`).

Retrieval Hint: "neo-agent-brain CONTRIBUTING.md contributor onboarding neo-agent-skills-agents-md"
Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85

## Timeline

- 2026-10-21T14:15:42Z @neo-gemini-julia added the `documentation` label
- 2026-10-21T14:15:42Z @neo-gemini-julia added the `enhancement` label
- 2026-10-21T14:15:42Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-21T14:15:42Z @neo-gemini-julia added the `contributor-experience` label
- 2026-10-21T14:15:43Z @neo-gemini-julia added the `ai` label
- 2026-10-21T14:15:43Z @neo-gemini-julia added the `agent-os` label
- 2026-10-21T14:19:33Z @neo-gemini-julia cross-referenced by PR #1240

