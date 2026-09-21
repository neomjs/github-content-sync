---
id: 47
title: 'The skills repo has no GitHub description and no topics, and three keywords under-describe it'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-04T13:53:10Z'
updatedAt: '2026-09-15T12:56:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/47'
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
closedAt: '2026-09-15T12:56:12Z'
---
# The skills repo has no GitHub description and no topics, and three keywords under-describe it

## Context

Operator sighting 2026-09-04: the org's GitHub metadata never followed the repository split. Measured the same day: `neomjs/neo-agent-skills` has **no repository description and zero topics**; its `package.json` carries a description ("The canonical agent skill substrate for the neomjs organization. Consumers depend on it; they never …") and three keywords (`neo.mjs`, `agent-skills`, `agent-os`). Sibling leaves exist for the engine (topics are the monorepo-era mix), the brain and the institution (both empty on GitHub).

Live latest-open sweep: no open issue mentions topics, keywords or the description in its title (2026-09-04T13:47Z). A2A claim sweep: none.

## The Problem

The skills repo is the substrate every other repo consumes (PR baseline, body lint, the agent skills); on GitHub it has no sentence and no topic, so neither a contributor nor a search reaches it. Three keywords under-describe what it is (skills, workflows, lint gates, the shared PR baseline).

## The Architectural Reality

- Description/topics: repository settings, admin write (`gh repo edit`) — the operator's hands. Limits: 20 topics, lowercase, letters/digits/hyphens, ≤ 50 chars.
- `package.json` keywords/description: a PR in this repo; the description sentence exists — reuse it on GitHub (one truth, two surfaces).

## The Fix

Description (proposal, from `package.json`): *The canonical agent skill substrate of the neomjs organization: the skills, workflows and lint gates every repo's agents run on — consumers depend on it, they never copy it.*

Topics (proposal, 14):

`ai, ai-agents, agent-os, agent-skills, multi-agent-systems, llm, prompt-engineering, developer-workflow, code-review, ci, lint, mcp, neo-mjs, javascript`

`package.json` keywords: extend to `neo-mjs, agent-skills, agent-os, ai-agents, agent-workflows, pr-review, code-review, lint-gates, multi-agent-systems, model-experience`.

```bash
gh repo edit neomjs/neo-agent-skills --description "The canonical agent skill substrate of the neomjs organization: the skills, workflows and lint gates every repo's agents run on — consumers depend on it, they never copy it."
gh repo edit neomjs/neo-agent-skills --add-topic ai --add-topic ai-agents --add-topic agent-os --add-topic agent-skills --add-topic multi-agent-systems --add-topic llm --add-topic prompt-engineering --add-topic developer-workflow --add-topic code-review --add-topic ci --add-topic lint --add-topic mcp --add-topic neo-mjs --add-topic javascript
```

## Acceptance Criteria

- [ ] AC-1 `gh repo view neomjs/neo-agent-skills --json description,repositoryTopics` shows the description and the topics (or the operator's edit, recorded in a closing comment).
- [ ] AC-2 A PR extends `package.json` keywords (and aligns its description with GitHub's).
- [ ] AC-3 The `neo.mjs` keyword becomes `neo-mjs` (the form the other repos use) — one spelling across the org.

## Out of Scope

- The other three repos (own leaves).

## Related

Sibling leaves (2026-09-04): neomjs/neo#18282 (engine: re-topic as the Body) · neomjs/neo-agent-brain#309 (brain: description + topics) · neomjs/neo-agent-institution#100 (institution: description + topics + package.json) · neomjs/neo-agent-skills#47 (skills: description + topics + keywords) · The engine, brain and institution leaves (same day) · `package.json`.

Ownership: unowned — the GitHub half is the operator's; the keywords PR is a two-line change any peer can take.

Origin Session ID: e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3

Retrieval Hint: `query_raw_memories("github topics repository description package.json keywords post-split engine brain institution skills")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session e1f9d3cb-6f4f-423c-9e42-6f20d9cba9b3


## Timeline

- 2026-09-04T13:53:11Z @neo-fable-clio added the `documentation` label
- 2026-09-04T13:53:12Z @neo-fable-clio added the `enhancement` label
- 2026-09-04T13:53:12Z @neo-fable-clio added the `ai` label
- 2026-09-04T13:53:59Z @neo-fable-clio cross-referenced by #18282
- 2026-09-04T13:54:01Z @neo-fable-clio cross-referenced by #309
- 2026-09-04T13:54:03Z @neo-fable-clio cross-referenced by #100
- 2026-09-04T16:30:08Z @neo-fable-clio cross-referenced by PR #105
- 2026-09-15T11:52:10Z @neo-opus-vega cross-referenced by PR #70

