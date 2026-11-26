---
id: 476
title: Include affected public repos in GitHub About checks
state: OPEN
labels:
  - documentation
  - enhancement
  - developer-experience
  - ai
  - needs-design-authority
assignees: []
createdAt: '2026-11-26T18:17:52Z'
updatedAt: '2026-11-26T18:18:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/476'
author: neo-gpt-sophie
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
# Include affected public repos in GitHub About checks

## Context

While checking Benchmarks on 26 November 2026, GitHub returned an empty description and topic list. Benchmarks already publishes web performance and coding-agent suites. A read-only check of six adjacent public repositories also found that `create-app` had a description and an empty topic list. [Benchmarks](https://github.com/neomjs/benchmarks) · [Create App](https://github.com/neomjs/create-app)

Benchmarks now has these 12 live topics: `benchmark`, `performance`, `web-performance`, `frontend`, `javascript`, `react`, `angular`, `playwright`, `data-grid`, `web-workers`, `ai-agents`, `neo-mjs`. GitHub rejected this seat’s description update through REST and GraphQL. The pending About description is:

> Reproducible web performance benchmarks for Neo.mjs, React, and Angular. Playwright tests for data grids and scrolling, plus a three-arm AI coding-agent benchmark.

## The Problem

The identity skill’s Class 4 inventory lists the five repositories covered by the earlier identity pass. Benchmarks and Create App sit outside the list. A reader following the inventory can complete its documented checks and miss both discovery surfaces. [Class 4](https://github.com/neomjs/neo-agent-skills/blob/5431cdadb00b21b5f561e68f10f5621b0ef232d6/.agents/skills/neo-identity-update/references/affected-areas-map.md#L36-L42)

## The Architectural Reality

ADR 0018 gives each repository its own product description and topics. Class 4 already owns the live read and the before/after record for GitHub metadata. The public repository’s About panel is the source for the deployed values. [ADR 0018](https://github.com/neomjs/neo/blob/3e43effbf34adf83d8436c7b93bff2e0236fa3e4/learn/agentos/decisions/0018-neo-identity-source-of-truth-model.md#L72-L81)

## The Fix

Update the existing Class 4 row to cover the affected public repositories when discovery metadata is in scope, including a newly introduced public repository. Keep the existing live metadata read. Use Benchmarks and Create App as examples of repositories outside the five covered by the original pass.

## Acceptance Criteria

- [ ] A reader can determine which public repositories to check without treating the five examples as an exhaustive list.
- [ ] The reference points to the existing `gh repo view neomjs/<repo> --json description,repositoryTopics` check and the before/after record or owner handoff.
- [ ] Exercise the guidance on Benchmarks and Create App. Record the observed description and topic values in the follow-up PR.

Decision Record impact: aligned-with Engine ADR 0018 §2.3.

## Out of Scope

Automated release checks, organization-wide metadata rewrites, and turn-loaded instructions.

## Related

Related: #175. Earlier five-repository pass: [Engine #19811](https://github.com/neomjs/neo/issues/19811).

Live latest-open + closed sweep: 20 latest open Skills issues, targeted GitHub issue and PR searches for About metadata and the inventory, plus Knowledge Base and Memory Core. Prior identity work covers the original five; no equivalent inventory follow-up surfaced.

unowned-rationale: Scoped documentation follow-up for a maintainer. Release scope remains independent.

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822

## Timeline

- 2026-11-26T18:17:54Z @neo-gpt-sophie added the `documentation` label
- 2026-11-26T18:17:54Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-26T18:17:54Z @neo-gpt-sophie added the `developer-experience` label
- 2026-11-26T18:17:54Z @neo-gpt-sophie added the `ai` label
- 2026-11-26T18:18:03Z @github-actions added the `needs-design-authority` label

