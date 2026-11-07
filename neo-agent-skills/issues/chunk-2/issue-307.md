---
id: 307
title: Add anti-underselling and capability-breadth gate to guide-authoring-bar.md
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gemini-julia
createdAt: '2026-11-07T03:24:45Z'
updatedAt: '2026-11-07T03:53:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/307'
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
# Add anti-underselling and capability-breadth gate to guide-authoring-bar.md

Design authority: N/A — non-behavior change (scope: docs)
Related: neomjs/neo#20496, neomjs/neo#20498, neomjs/neo#20519, neomjs/neo#20538, neomjs/neo#20545

## Context

In `neomjs/neo#20538` / `neomjs/neo#20545`, `.agents/skills/guide-authoring/references/guide-authoring-bar.md` was updated to codify the anti-underselling, `src/` capability-breadth (including `src/dashboard/dock/` multi-window dock layouts), JSON-first LLM fluency (since the Nov 2019 GA), and no-app-concession gate. Because `.agents/skills/**` in `neomjs/neo` is materialized from `neomjs/neo-agent-skills`, `neomjs/neo#20545` Review `5809724064` (RA-3) requires landing this update in `neomjs/neo-agent-skills` first so `postinstall` (`neo-agent-skills-materialize`) does not overwrite the local Engine copy.

## Acceptance Criteria

- [ ] **AC-1 (`guide-authoring-bar.md` `§2` & `§6` + `framing-governance.md`):** Add the repository-agnostic anti-underselling, positioning-page capability-breadth (`src/grid/`, `src/form/field/`, `src/calendar/`, `src/dashboard/dock/` multi-window dock layouts, `src/layout/`, `src/main/addon/`), JSON-first LLM fluency, and no-app-concession gate to `.agents/skills/guide-authoring/references/guide-authoring-bar.md` (`§2` and `§6`) with a retirement trigger, and add a one-line cross-reference in `.agents/skills/neo-identity-update/references/framing-governance.md`.
- [ ] **AC-2 (Budget & Corpus Lint):** Verify `.agents/skills/guide-authoring/references/guide-authoring-bar.md` and `.agents/skills/neo-identity-update/references/framing-governance.md` pass `node scripts/lint-skill-corpus.mjs --base origin/dev` cleanly (`40 skills, coherent with the manifest, within budget, document reach canonical`).
- [ ] **AC-3 (Version Bump):** Bump `package.json` and `package-lock.json` patch version (`0.1.95` → `0.1.96`) and verify `node scripts/check-version-bump.mjs --base origin/dev` passes.

## Timeline

- 2026-11-07T03:24:46Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-07T03:24:47Z @neo-gemini-julia added the `documentation` label
- 2026-11-07T03:24:47Z @neo-gemini-julia added the `enhancement` label
- 2026-11-07T03:24:47Z @neo-gemini-julia added the `ai` label
- 2026-11-07T03:24:47Z @neo-gemini-julia added the `model-experience` label
- 2026-11-07T03:27:00Z @neo-gemini-julia cross-referenced by PR #308
- 2026-11-07T03:29:00Z @neo-gemini-julia cross-referenced by #20538
- 2026-11-07T03:29:01Z @neo-gemini-julia cross-referenced by PR #20545
- 2026-11-07T03:42:39Z @neo-gemini-julia referenced in commit `015d82b` - "fix(skills): address review RAs on guide-authoring-bar and lockfile [skill-growth-justified: codify #20498 front-door anti-concession gate with retirement trigger] (#307)"

