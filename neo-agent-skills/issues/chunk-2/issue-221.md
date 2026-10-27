---
id: 221
title: 'Teach detail: ''summary'' and agentIdentity on query_raw_memories sweeps in memory-mining-protocol.md'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-27T18:57:51Z'
updatedAt: '2026-10-27T20:06:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/221'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 11
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-27T20:06:38Z'
---
# Teach detail: 'summary' and agentIdentity on query_raw_memories sweeps in memory-mining-protocol.md

## Context

Companion `neomjs/neo-agent-skills` leaf split from `neomjs/neo-agent-brain#1290` [QUARANTINED_URL: github.com] (`AC-2`) per `pull-request-workflow.md §9.1` (one repository-local `Resolves #N` per PR), following the merge of `neomjs/neo-agent-brain#1288` [QUARANTINED_URL: github.com] (`#1280` — `agentIdentity`, `detail: 'summary' | 'full'`, and `projection: 'public' | 'private'` on `query_raw_memories`, plus `agentIdentity` on `query_summaries`). Grounded in `pullrequestreview-5646779645` [QUARANTINED_URL: github.com] `RA-5` by `@neo-opus-ada` and `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com], dispositioned by `@neo-fable-clio` in `comment 19184558` [QUARANTINED_URL: github.com]).

## The Problem

`neomjs/neo-agent-brain#1280` (`PR #1288`) keeps `detail: 'full'` as the default on `query_raw_memories` so existing callers and unit specs remain 100% backward compatible. Consequently, the 30k–80k char context-window savings across a 3–10 call pre-implementation / pre-PR-review sweep only materialize when `.agents/skills/memory-mining/references/memory-mining-protocol.md` teaches agents to pass `detail: 'summary'` (and `agentIdentity` when author-scoped) on initial discovery sweeps and reserve `detail: 'full'` (`projection: 'public'` on cross-seat team reads) for hydrating matching hits. Today, `.agents/skills/memory-mining/references/memory-mining-protocol.md` `§2 Tool order by freshness` (`lines 38-41`) still points at `query_raw_memories` without `detail: 'summary'` or `agentIdentity`.

## The Architectural Reality

- `.agents/skills/memory-mining/references/memory-mining-protocol.md` (`lines 34-42` in `neomjs/neo-agent-skills`) owns `§2 Tool order by freshness`.
- Top-level `.agents/skills/memory-mining/SKILL.md` (`8` lines, `674` bytes) is a progressive-disclosure router and remains untouched (`0` bytes delta on always-loaded surfaces).
- Every non-Dependabot PR in `neomjs/neo-agent-skills` requires a patch version bump in `package.json` and `package-lock.json` (`check-version-bump.mjs`).

Structure-map gate: `npm run ai:structure-map -- --files --loc` run `2026-10-26T02:43Z` in `neomjs/neo-agent-brain`; `.agents/skills/memory-mining/references/memory-mining-protocol.md` owns the skill reference surface in `neomjs/neo-agent-skills`.

## The Fix

1. Rewrite `.agents/skills/memory-mining/references/memory-mining-protocol.md` `§2 Tool order by freshness` (`lines 38-41`) so:
   - Item 1 (`Recent / active-session investigations`) teaches running `query_raw_memories` (`detail: 'summary'`, plus `agentIdentity` when author-scoped) first or in parallel with `query_summaries`.
   - Item 2 (`Older / stable history`) notes `(with agentIdentity when author-scoped)` on `query_summaries`.
   - Item 3 (`query_raw_memories (detail: 'full') after a summary hit`) teaches hydrating matching turns (`projection: 'public'` on cross-seat team reads) to recover turn-level reasoning, commands, message IDs, or decision texture.
   - Tighten surrounding prose in `§2` so the edit is net-negative in bytes (`-248` bytes on `memory-mining-protocol.md`).
2. Bump `version` in `package.json` and `package-lock.json` from `0.1.64` to `0.1.65`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `.agents/skills/memory-mining/references/memory-mining-protocol.md` (`§2 Tool order by freshness`, `lines 38-41`) | `neomjs/neo-agent-brain#1280` (`PR #1288`) + `neomjs/neo` Discussion `#20111` Option F | Teaches `detail: 'summary'` (and `agentIdentity`) on initial `query_raw_memories` discovery sweeps and `detail: 'full'` (`projection: 'public'` on cross-seat reads) for hit hydration, net-negative in bytes (`-248` bytes) | Existing `query_raw_memories` calls without `detail` continue to default to `'full'` | `memory-mining-protocol.md` | `npm run lint` + `npm test` (`40/40` corpus + all contract runners green) |

## Decision Record impact

`aligned-with` `neomjs/neo-agent-brain#adr:agents-md-compaction-taxonomy` (`0007-agents-md-compaction-taxonomy.md`) — `0` bytes added to `AGENTS.md` or `SKILL.md`; `-248` bytes net-negative rewrite in on-demand `memory-mining-protocol.md`.

## Acceptance Criteria

- [ ] `.agents/skills/memory-mining/references/memory-mining-protocol.md` (`§2 Tool order by freshness`) teaches `detail: 'summary'` (and `agentIdentity` when author-scoped) for initial `query_raw_memories` sweeps and `detail: 'full'` (`projection: 'public'` on cross-seat team reads) for hit hydration, with net-non-positive byte delta (`-248` bytes).
- [ ] `package.json` and `package-lock.json` bump `version` to `0.1.65`, and `npm run lint` + `npm test` pass.

## Out of Scope

- Brain-side `seatMemoryLayerTemplate.mjs`, `SummaryService.mjs`, and L3 live-plane `explore_memory_history` verification (tracked and delivered in `neomjs/neo-agent-brain#1290` [QUARANTINED_URL: github.com]).

## Avoided Traps

- **Adding guidance to top-level `SKILL.md` or `AGENTS.md`:** Keeps `0` bytes delta on session-boot routers; places parameter detail in the on-demand reference payload while net-reducing its byte size (`-248` bytes).

## Related

- Split from `neomjs/neo-agent-brain#1290` [QUARANTINED_URL: github.com] (`AC-2`)
- Follow-up to `neomjs/neo-agent-brain#1280` [QUARANTINED_URL: github.com] (`PR #1288` [QUARANTINED_URL: github.com] `RA-5` by `@neo-opus-ada`)
- `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-skills` at `2026-10-27T18:57:14Z` (newest `#220`); no duplicate found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 30 })` at `2026-10-27T18:56:47Z`; only our own `[lane-claim]` (`MESSAGE:c5332089-6ab8-4770-ad4a-3751c029b752`) covers this scope.
MC sweep: `query_raw_memories` on `"query_raw_memories 30k 80k context window bloat memory-mining-protocol"`, 6 results, no prior `neomjs/neo-agent-skills` issue found.
Own-assignment sweep: `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-skills`.

Origin Session ID: 58b07b45-1fe9-40ad-80f8-db0f380332b8
Retrieval Hint: "memory-mining-protocol query_raw_memories detail summary agentIdentity projection public 1280 1288 1290"


## Timeline

- 2026-10-27T18:57:52Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-27T18:57:53Z @neo-gemini-maryam added the `documentation` label
- 2026-10-27T18:57:54Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-27T18:57:54Z @neo-gemini-maryam added the `ai` label
- 2026-10-27T18:57:54Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-27T18:57:55Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-27T18:58:35Z @neo-gemini-maryam cross-referenced by #1290
- 2026-10-27T19:00:17Z @neo-gemini-maryam cross-referenced by PR #222
- 2026-10-27T19:00:28Z @neo-gemini-maryam cross-referenced by PR #1320
- 2026-10-27T20:06:39Z @tobiu closed this issue
- 2026-10-27T20:06:39Z @tobiu referenced in commit `c7a1089` - "docs(memory-mining): teach detail summary and agentIdentity on raw sweeps (#221) (#222)"

