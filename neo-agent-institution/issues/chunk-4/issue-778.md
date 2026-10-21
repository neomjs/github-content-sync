---
id: 778
title: Add CONTRIBUTING.md for human and autonomous-agent contributors
state: CLOSED
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
  - contributor-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-10-21T14:15:45Z'
updatedAt: '2026-10-21T14:37:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/778'
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
closedAt: '2026-10-21T14:37:37Z'
---
# Add CONTRIBUTING.md for human and autonomous-agent contributors

## Context

Following the external-agent contribution on `neomjs/neo#19952` / `neomjs/neo#19959` (`@Jah-yee`), `@tobiu` noted that while `neomjs/neo` has a root [`CONTRIBUTING.md`](https://github.com/neomjs/neo/blob/dev/CONTRIBUTING.md) covering local setup, test tiers, issue claiming (`good first issue` / `help wanted`), and autonomous-agent onboarding (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo <repo> --audience contributor`), `neomjs/neo-agent-institution` does not yet have a `CONTRIBUTING.md`.

## The Problem

1. External human and autonomous-agent contributors arriving at `neomjs/neo-agent-institution` have no root `CONTRIBUTING.md` explaining:
   - the difference between running the Fleet Manager cockpit vs changing the Agent Institution product (`apps/agentos/`, `src/`, `resources/scss/`, `harness/`, `test/`);
   - how to clone, install (`npm install`), start the browser app (`npm run server-start`), recompile themes after `.scss` edits (`npm run build-themes` / `npm run watch-themes`), and run the isolated test tiers (`npm run test-unit`, `npm run test-components`, `npm run test-e2e`, `npm run test-visual`) without needing a Brain checkout, Docker, or maintainer credentials;
   - why `npm run test-*` must be used instead of bare `npx playwright test` (each tier has its own config under `test/playwright/`);
   - when `NEO_AGENTOS_RUNTIME_ROOT=/absolute/path/to/neo-agent-brain` is needed (`npm run test-e2e:nl` and cross-repo contract runs);
   - how to pick up `good first issue` / `help wanted` issues, claim before starting, target `dev` (not `main`), and run `npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-institution --audience contributor` for contributor-scoped agent instructions.
2. In `README.md`, the `PRs Welcome` badge (`:12`) and `## Contributing` section (`:198-202`) do not yet link to `./CONTRIBUTING.md`.

## The Architectural Reality

- `package.json` defines `"postinstall": "neo-agent-skills-materialize"`, `"server-start"`, `"build-themes"`, `"watch-themes"`, `"test-unit"`, `"test-components"`, `"test-e2e"`, `"test-visual"`, and `"test-e2e:nl"`.
- `neo-agent-skills` (`scripts/generate-agents-md.mjs`) already supports `--repo neo-agent-institution --audience contributor`.
- Every relative link inside `CONTRIBUTING.md` must resolve to an actual file in `neomjs/neo-agent-institution` (such as `README.md`, `ROADMAP.md`, `learn/README.md`, `harness/README.md`, `LICENSE`) or use an explicit canonical URL when referencing sibling repositories.

## The Fix

1. Author a root `CONTRIBUTING.md` in `neomjs/neo-agent-institution` modeled on `neomjs/neo/CONTRIBUTING.md` and tailored to Agent Institution's exact setup, SCSS theme build workflow, isolated vs Brain-connected test tiers, issue-claiming workflow (`good first issue` / `help wanted`, `dev` target branch), and AI agent contributor section (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-institution --audience contributor`).
2. Update `README.md` (`PRs Welcome` badge and `## Contributing` section) to link to `./CONTRIBUTING.md`.

## Decision Record impact

`none` — documentation and contributor-onboarding alignment with `neomjs/neo/CONTRIBUTING.md`.

## Acceptance Criteria

- [ ] AC-1: Root `CONTRIBUTING.md` exists in `neomjs/neo-agent-institution` covering starting points, zero-credential clone + `npm install` + `npm run server-start` + `npm run test-unit` / `test-components` / `test-e2e` / `test-visual` setup, `.scss` theme compilation (`build-themes` / `watch-themes`), why bare `npx playwright test` fails without `-c`, what `npm install` (`postinstall`) materializes, issue claiming (`good first issue` / `help wanted`, `dev` target branch), and AI agent contributor instructions (`npx -p neo-agent-skills neo-agent-skills-agents-md --repo neo-agent-institution --audience contributor`).
- [ ] AC-2: Every relative link in `CONTRIBUTING.md` and `README.md` resolves to an existing path in `neomjs/neo-agent-institution` (verified mechanically).
- [ ] AC-3: `README.md` links the `PRs Welcome` badge and `## Contributing` section to `./CONTRIBUTING.md`.

## Out of Scope

- Adding `CONTRIBUTING.md` to `neomjs/neo-agent-brain` (tracked in its sibling `neomjs/neo-agent-brain` ticket).

## Related

- `neomjs/neo/CONTRIBUTING.md`
- `neomjs/neo#19952`, `neomjs/neo#19959`

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` at 2026-10-21T14:13Z; no equivalent found.
MC sweep: `neo-agent-brain neo-agent-institution CONTRIBUTING.md outside contributor setup loop 0101-orientation-contributor`, 10 results, no prior decision found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-institution`, none overlapping.
Structure map: N/A (documentation / contributor onboarding in `neomjs/neo-agent-institution`).

Retrieval Hint: "neo-agent-institution CONTRIBUTING.md contributor onboarding neo-agent-skills-agents-md"
Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85

## Timeline

- 2026-10-21T14:15:47Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-21T14:15:47Z @neo-gemini-julia added the `documentation` label
- 2026-10-21T14:15:47Z @neo-gemini-julia added the `enhancement` label
- 2026-10-21T14:15:48Z @neo-gemini-julia added the `agent-os` label
- 2026-10-21T14:15:48Z @neo-gemini-julia added the `ai` label
- 2026-10-21T14:15:48Z @neo-gemini-julia added the `contributor-experience` label
- 2026-10-21T14:19:16Z @neo-gemini-julia cross-referenced by PR #779
- 2026-10-21T14:37:38Z @tobiu closed this issue
- 2026-10-21T14:37:38Z @tobiu referenced in commit `550f8bb` - "docs(contributing): add CONTRIBUTING.md for human and agent contributors (#778) (#779)"

