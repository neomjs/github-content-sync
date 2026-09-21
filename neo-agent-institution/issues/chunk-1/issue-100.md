---
id: 100
title: 'The Institution repo has no GitHub description, no topics, and no package.json description or keywords'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-04T13:53:08Z'
updatedAt: '2026-09-04T17:14:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/100'
author: neo-fable-clio
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
closedAt: '2026-09-04T17:14:39Z'
---
# The Institution repo has no GitHub description, no topics, and no package.json description or keywords

## Context

Operator sighting 2026-09-04: the org's GitHub metadata never followed the repository split. Measured the same day: `neomjs/neo-agent-institution` has **no repository description, zero topics, and a `package.json` with neither `description` nor `keywords`** — the emptiest of the four repos (engine: 20 monorepo-era topics; brain: 35 keywords, no GitHub metadata; skills: 3 keywords, no GitHub metadata). Sibling leaves exist for the other three.

Live latest-open sweep: no open issue mentions topics, keywords or the description in its title (2026-09-04T13:47Z). A2A claim sweep: none.

## The Problem

This is the product repository — the Fleet Manager cockpit an operator downloads and runs — and on GitHub it has no sentence and no topic. Nothing on the org page says what it is; no topic page reaches it; `npm view` shows no description. The 24-ticket product backlog and the release video (#19) point people here.

## The Architectural Reality

- GitHub description/topics: repository settings (admin write; `gh repo edit`) — the operator's hands. Limits: 20 topics, lowercase, letters/digits/hyphens, ≤ 50 chars.
- `package.json` `description` + `keywords`: a PR in this repo (`package.json` is a lock/stamp-neutral edit; no visual baseline input).
- Vocabulary source: README.md's opening and the cockpit's own words — fleet cockpit, agent institution, Agent OS harness, dock layouts, multi-window, Neural Link, Electron shell (#7).

## The Fix

Description (proposal): *The Agent Institution of Neo.mjs: the Fleet Manager cockpit that hosts, observes and operates a cross-model agent swarm — dock layouts, multi-window, Neural Link, and the harness a fleet boots from.*

Topics (proposal, 18):

`ai, ai-agents, agent-os, multi-agent-systems, fleet-management, agent-fleet, cockpit, dashboard, dock-layout, multi-window, neural-link, mcp, javascript, web-workers, sharedworker, electron, pwa, neo-mjs`

`package.json`: add `"description"` (the sentence above) and `"keywords"` derived from the topics plus the product's own terms (`fleet-manager`, `agent-institution`, `agent-cockpit`, `agent-harness`, `dock-layouts`, `multi-window-applications`, `neural-link`, `possession-interface`, `agent-os`, `ai-native`).

```bash
gh repo edit neomjs/neo-agent-institution --description "The Agent Institution of Neo.mjs: the Fleet Manager cockpit that hosts, observes and operates a cross-model agent swarm — dock layouts, multi-window, Neural Link, and the harness a fleet boots from."
gh repo edit neomjs/neo-agent-institution --add-topic ai --add-topic ai-agents --add-topic agent-os --add-topic multi-agent-systems --add-topic fleet-management --add-topic agent-fleet --add-topic cockpit --add-topic dashboard --add-topic dock-layout --add-topic multi-window --add-topic neural-link --add-topic mcp --add-topic javascript --add-topic web-workers --add-topic sharedworker --add-topic electron --add-topic pwa --add-topic neo-mjs
```

## Acceptance Criteria

- [ ] AC-1 `gh repo view neomjs/neo-agent-institution --json description,repositoryTopics` shows the description and the topic list (or the operator's edit, recorded in a closing comment).
- [ ] AC-2 A PR adds `description` and `keywords` to `package.json`; `npm run test-unit` untouched; the two descriptions (npm, GitHub) say the same thing.
- [ ] AC-3 README.md's opening sentence and the description agree in vocabulary (edit whichever drifted; no new prose).

## Out of Scope

- The other three repos (own leaves).
- Homepage URL (there is no product site yet; #19's video is the first public surface).

## Related

Sibling leaves (2026-09-04): neomjs/neo#18282 (engine: re-topic as the Body) · neomjs/neo-agent-brain#309 (brain: description + topics) · neomjs/neo-agent-institution#100 (institution: description + topics + package.json) · neomjs/neo-agent-skills#47 (skills: description + topics + keywords) · #10 (parent arc) · #7 (Electron shell — the `electron` topic is a promise, remove it if the shell is dropped) · #19 (release video) · the engine, brain and skills leaves (same day).

Ownership: the `package.json` half is a five-line PR any peer can take; the GitHub half is the operator's. Unowned at filing.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("github topics repository description package.json keywords post-split engine brain institution skills")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-09-04T13:53:09Z @neo-fable-clio added the `documentation` label
- 2026-09-04T13:53:10Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T13:53:10Z @neo-fable-clio added the `ai` label
- 2026-09-04T13:53:59Z @neo-fable-clio cross-referenced by #18282
- 2026-09-04T13:54:01Z @neo-fable-clio cross-referenced by #309
- 2026-09-04T13:54:04Z @neo-fable-clio cross-referenced by #47
- 2026-09-04T16:28:23Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-04T16:30:08Z @neo-fable-clio cross-referenced by PR #105
- 2026-09-04T17:14:39Z @tobiu referenced in commit `ce47818` - "Merge pull request #105 from neomjs/agent/100-package-metadata

chore(institution): package.json gains the product description and keywords the split left empty (#100)"
- 2026-09-04T17:14:39Z @tobiu closed this issue

