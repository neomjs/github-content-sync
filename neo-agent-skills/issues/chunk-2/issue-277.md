---
id: 277
title: Two skill pages name npm scripts that Brain retired
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - needs-design-authority
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T13:51:31Z'
updatedAt: '2026-11-05T15:52:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/277'
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
closedAt: '2026-11-05T15:52:35Z'
---
# Two skill pages name npm scripts that Brain retired

## Context

A sweep on 2026-11-05 checked every `npm run <script>` the corpus names: 142 pages, 14 distinct names. Each was checked against the `package.json` of neo, Brain, Institution and this repository. After neomjs/neo-agent-brain#1482 added `ai:audit-discussion-lifecycle`, two names remain that no repository defines. Both scripts existed once, and both were retired from Brain on purpose:

| Name | Named at | History |
|---|---|---|
| `ai:run-sandman` | `session-sunset/references/session-sunset-workflow.md:97` | neo `e64b68ee3e` (#9705) → Brain with the Agent OS (neomjs/neo-agent-brain#13) → dropped from Brain's `package.json` in neomjs/neo-agent-brain#197. The runner is still `ai/scripts/runners/runSandman.mjs`. |
| `ai:lint-tree-json` | `guide-authoring/references/guide-authoring-bar.md:56` | neo `699ee65567` (#12248) → Brain → `npm` entry dropped in neomjs/neo-agent-brain#214, file deleted with the engine projections in neomjs/neo-agent-brain#198. No copy exists in neo. |

## The Problem

- **The sunset page names a command that cannot run, inside a prohibition.** "Agents must NOT trigger sandman (`npm run ai:run-sandman` / `GoldenPathSynthesizer`)". The rule is right, but the command it forbids no longer exists, so a reader matching on what they are about to run never sees their own invocation.
- **The guide bar promises a check that no longer exists.** "`learn/tree.json` — the nav SSOT (`npm run ai:lint-tree-json` green when present)". Nothing validates `tree.json`'s structure today. I ran the retired lint (Brain `5c8e70a7`) against neo `dev`. It reported a route leaf, `ApiDocs`, which is intentional (#20279), and a stale call into the SEO generator, which now needs `--corpus-root`. No real drift. So the retirement has no measured cost, and the page should stop naming the command.

## The Fix

- `session-sunset-workflow.md:97`: name what exists: `ai/scripts/runners/runSandman.mjs` / `GoldenPathSynthesizer`.
- `guide-authoring-bar.md:56`: drop the dead command. `tree.json` stays the SSOT the author keeps in sync.

## Acceptance Criteria

- [ ] AC-1: Neither page names `npm run ai:run-sandman` or `npm run ai:lint-tree-json`. A re-run of the sweep finds every `npm run` name in the corpus defined in one of the four `package.json` files.
- [ ] AC-2: `npm run lint` passes in this repository.

## Out of Scope

- Restoring a `tree.json` structure lint in neo. There is no measured drift, and the retired lint would need two updates (route leaves, `--corpus-root`) first.
- Any change to the sandman prohibition itself.

## Related

neomjs/neo-agent-brain#1482 (the third name, fixed) · neomjs/neo-agent-brain#197 · neomjs/neo-agent-brain#198 · neomjs/neo-agent-brain#214

Live latest-open + closed sweep: the latest 20 open issues here, plus searches (`ai:run-sandman`, `ai:lint-tree-json`, "npm script skill missing"), at 2026-11-05T13:51Z. No equivalent. A2A claims in the last hour: none on these pages; Grace's sunset-reinstall work moved to neomjs/neo-agent-brain#1480. Own open assignments: no same surface.

Origin Session ID: 57180f73-f0b4-4ae5-bac8-095fd4813333

Retrieval Hint: "skill pages name retired npm scripts ai:run-sandman ai:lint-tree-json"

Authored by Vega (Opus 5.5, Claude Code)

## Timeline

- 2026-11-05T13:51:31Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T13:51:33Z @neo-opus-vega added the `bug` label
- 2026-11-05T13:51:33Z @neo-opus-vega added the `documentation` label
- 2026-11-05T13:51:34Z @neo-opus-vega added the `ai` label
- 2026-11-05T13:52:02Z @github-actions added the `needs-design-authority` label
- 2026-11-05T13:52:50Z @neo-opus-vega cross-referenced by PR #278
- 2026-11-05T14:04:31Z @neo-opus-vega referenced in commit `2d986a5` - "build(release): neo-agent-skills 0.1.89 carries the two corrected pages (#277)"
- 2026-11-05T15:52:35Z @tobiu referenced in commit `c9e0e07` - "fix(skills): two pages stop naming npm scripts Brain retired (#277) (#278)

* fix(skills): two pages stop naming npm scripts Brain retired (#277)

* build(release): neo-agent-skills 0.1.89 carries the two corrected pages (#277)"
- 2026-11-05T15:52:36Z @tobiu closed this issue

