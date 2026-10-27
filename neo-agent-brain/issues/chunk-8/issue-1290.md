---
id: 1290
title: 'Teach detail: ''summary'' and agentIdentity on query_raw_memories sweeps in seat memory and skills'
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-26T02:46:41Z'
updatedAt: '2026-10-27T20:07:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1290'
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
blockedBy:
  - '[x] 1280 Support agentIdentity and summary detail on query_raw_memories and query_summaries'
blocking: []
closedAt: '2026-10-27T20:07:14Z'
---
# Teach detail: 'summary' and agentIdentity on query_raw_memories sweeps in seat memory and skills

## Context

Follow-up leaf to `neomjs/neo-agent-brain#1280` (`PR #1288`) and `neomjs/neo-agent-brain#1281` (`PR #1285`), requested in Cycle-1 reviews (`pullrequestreview-5646779645` [QUARANTINED_URL: github.com] `RA-5` by `@neo-opus-ada` and `pullrequestreview-5647022042` [QUARANTINED_URL: github.com] `RA-4` by `@neo-opus-grace`) and grounded in `neomjs/neo` Discussion `#20111` (comment `19184520` [QUARANTINED_URL: github.com], dispositioned by `@neo-fable-clio` in `comment 19184558` [QUARANTINED_URL: github.com]).

## The Problem

1. **Teaching `detail: 'summary'` and `agentIdentity` on `query_raw_memories` discovery sweeps (`PR #1288` `RA-5`):**
   - `neomjs/neo-agent-brain#1280` (`PR #1288`) adds `agentIdentity`, `detail: 'summary' | 'full'`, and `projection: 'public' | 'private'` to `query_raw_memories` (and `agentIdentity` to `query_summaries`), keeping `detail: 'full'` as the default on `query_raw_memories` so existing callers and unit specs remain 100% backward compatible.
   - Keeping `detail: 'full'` as the default means the 30k–80k char context-window savings only materialize when agents and skills pass `detail: 'summary'` during discovery sweeps and reserve `detail: 'full'` for hydrating specific hits. Today, `ai/services/fleet/seatMemoryLayerTemplate.mjs:99-100,157,176,206` still points at `query_raw_memories` without `detail: 'summary'` (the companion `.agents/skills/memory-mining/references/memory-mining-protocol.md` surface is split to `neomjs/neo-agent-skills#221` [QUARANTINED_URL: github.com] per `pull-request-workflow.md §9.1`).
2. **Neutralizing `SummaryService.resolveAuthorScope` error prefix (`PR #1288` Round-2 note):**
   - `SummaryService.resolveAuthorScope` (`ai/services/memory-core/SummaryService.mjs:94`) starts its error string with `"SummaryService.listSummaries:"` even when called from `query_raw_memories` or `query_summaries`.
3. **Post-Merge L3 Live-Plane Latency Verification for `explore_memory_history` (`PR #1285` `RA-4` `Residual-Owner: #1290`):**
   - `neomjs/neo-agent-brain#1281` (`PR #1285`) bounds `explore_memory_history` via parallel per-identity turn walks and a 15s synthesis timeout (`L1` unit verified). Once `#1285` deploys to the Memory Core container plane, verify one live `explore_memory_history` call over the incident's window completes under the 60s MCP deadline (`L3`).

## The Architectural Reality

- `ai/services/fleet/seatMemoryLayerTemplate.mjs` (`410` LOC in `neomjs/neo-agent-brain`):
  - Line `100` renders the `MEMORY.md` header note pointing to `query_raw_memories` and `query_recent_turns`; lines `157`, `176`, and `206` render `about-this-layer.md` across Antigravity, Codex, and Claude/OpenCode/Kimi harnesses.
  - `test/playwright/unit/ai/services/fleet/seatMemoryLayerTemplate.spec.mjs` tests the rendered template output; `generateKimiSeatConfig.spec.mjs` and `generateOpenCodeSeatConfig.spec.mjs` pin the SHA-256 artifact digest of generated seat configs.
- `ai/services/memory-core/SummaryService.mjs` (`line 94`):
  - `resolveAuthorScope` error prefix, tested in `test/playwright/unit/ai/services/memory-core/SummaryService.AuthorScope.spec.mjs`.
- `.agents/skills/memory-mining/references/memory-mining-protocol.md` (canonical in `neomjs/neo-agent-skills`):
  - Split to companion leaf `neomjs/neo-agent-skills#221` [QUARANTINED_URL: github.com] so each repository's PR carries a single repository-local `Resolves #N`.

Structure map (`npm run ai:structure-map -- --files --loc` run `2026-10-26T02:43Z`): `ai/services/fleet/seatMemoryLayerTemplate.mjs` and `ai/services/memory-core/SummaryService.mjs` own the Brain-side surfaces.

## The Fix

1. **In `neomjs/neo-agent-brain` (`ai/services/fleet/seatMemoryLayerTemplate.mjs` & `SummaryService.mjs`):**
   - Update the Memory Core pointer in `renderMemoryIndexMd` (`line 100`) and `renderAboutThisLayerMd` (`lines 157, 176, 206`) so generated seat memory layers teach `query_raw_memories` with `detail: 'summary'` (plus `agentIdentity` when author-scoped) for discovery sweeps before `detail: 'full'` hydration.
   - Update `SummaryService.resolveAuthorScope` (`line 94`) error prefix from `"SummaryService.listSummaries:"` to `"SummaryService.resolveAuthorScope:"`.
   - Update `test/playwright/unit/ai/services/fleet/seatMemoryLayerTemplate.spec.mjs`, `test/playwright/unit/ai/services/memory-core/SummaryService.AuthorScope.spec.mjs`, and the SHA-256 digest pins in `generateKimiSeatConfig.spec.mjs` and `generateOpenCodeSeatConfig.spec.mjs`.
2. **Post-Deploy L3 Verification (`Residual-Owner` for `#1281` / `PR #1285`):**
   - Run one live `explore_memory_history` call on the deployed Memory Core plane over a 7d window and record the `< 60s` completion receipt.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ai/services/fleet/seatMemoryLayerTemplate.mjs` (`lines 99-100, 157, 176, 206`) | `neomjs/neo-agent-brain#1280` (`PR #1288`) + Discussion `#20111` Option F | Teaches `detail: 'summary'` (plus `agentIdentity` when author-scoped) on `query_raw_memories` discovery sweeps before `detail: 'full'` hydration | Existing `MEMORY.md` files remain valid | `seatMemoryLayerTemplate.mjs` JSDoc + rendered markdown | `seatMemoryLayerTemplate.spec.mjs`, `generateKimiSeatConfig.spec.mjs`, `generateOpenCodeSeatConfig.spec.mjs` |
| `SummaryService.resolveAuthorScope` (`SummaryService.mjs:94`) | `PR #1288` Round-2 review note | Uses neutral `"SummaryService.resolveAuthorScope:"` prefix on unbound `@me` error | Throws `Error` as before | `SummaryService.mjs` JSDoc | `SummaryService.AuthorScope.spec.mjs` |
| `explore_memory_history` live plane call (`Residual-Owner` for `#1281` / `PR #1285`) | `neomjs/neo-agent-brain#1281` (`PR #1285` `RA-4`) | Completes under the 60s MCP deadline on the deployed plane | Degrades synthesis via `synthesis-failed: timeout` at 15s | `PR #1285` Post-Merge Validation | L3 live plane receipt (`21s` weekly / `40s` daily) |

## Decision Record impact

`aligned-with` `learn/agentos/decisions/0007-agents-md-compaction-taxonomy.md` — any `AGENTS.md` touch must stay net-neutral or net-negative in loaded bytes; primary procedural detail lives in `seatMemoryLayerTemplate.mjs` and `memory-mining-protocol.md` (`neomjs/neo-agent-skills#221`).

## Acceptance Criteria

- [ ] `ai/services/fleet/seatMemoryLayerTemplate.mjs` teaches `detail: 'summary'` (plus `agentIdentity` when author-scoped) on `query_raw_memories` discovery sweeps, and `SummaryService.resolveAuthorScope` uses a neutral error prefix, with `seatMemoryLayerTemplate.spec.mjs` and `SummaryService.AuthorScope.spec.mjs` passing.
- [ ] ~~`memory-mining-protocol.md` (`§2 Tool order by freshness`) teaches `detail: 'summary'` for initial `query_raw_memories` sweeps and `detail: 'full'` (`projection: 'public'` for cross-seat reads) for hit hydration.~~ *(Split to companion repository-local leaf `neomjs/neo-agent-skills#221` [QUARANTINED_URL: github.com] per `pull-request-workflow.md §9.1`)*
- [ ] Post-deploy L3 residual from `#1281` / `PR #1285`: one `explore_memory_history` call on the live Memory Core plane over the incident's window completes under 60s.

## Out of Scope

- Changing `query_raw_memories`'s schema default from `detail: 'full'` to `detail: 'summary'` (intentionally kept `'full'` in `#1280` to preserve backward compatibility for existing callers/tests).
- `.agents/skills/memory-mining/references/memory-mining-protocol.md` in `neomjs/neo-agent-skills` (split to `neomjs/neo-agent-skills#221` [QUARANTINED_URL: github.com]).

## Avoided Traps

- **Bloating `AGENTS.md` loaded byte count:** Detailed parameter guidance belongs in `memory-mining-protocol.md` (`neomjs/neo-agent-skills#221`) and `seatMemoryLayerTemplate.mjs`, not in turn-loaded `AGENTS.md` prose (`ADR-0007`).

## Related

- Companion skills leaf: `neomjs/neo-agent-skills#221` [QUARANTINED_URL: github.com]
- Follow-up to `neomjs/neo-agent-brain#1280` (`PR #1288` Cycle-1 review `pullrequestreview-5646779645` `RA-5` and Round-2 review `pullrequestreview-5646882329` by `@neo-opus-ada`)
- Residual owner (`Residual-Owner: #1290`) for `neomjs/neo-agent-brain#1281` (`PR #1285` Cycle-1 review `pullrequestreview-5647022042` `RA-4` by `@neo-opus-grace`)
- `neomjs/neo` Discussion `#20111` (Option F — comment `19184520` [QUARANTINED_URL: github.com], disposition `19184558` [QUARANTINED_URL: github.com])

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-26T02:43Z` (newest `#1286`); no duplicate found.
A2A in-flight claim sweep: checked `list_messages({ status: 'all', limit: 15 })` at `2026-10-26T02:43Z`; no competing claim.
MC sweep: `query_raw_memories` on `query_raw_memories summary seatMemoryLayerTemplate`, 5 results, no prior teaching ticket found.
Own-assignment sweep: `#1280` and `#1281` currently assigned in `neomjs/neo-agent-brain`.

Origin Session ID: 06cf8283-28b4-42b3-b137-ae9da8ab6995
Retrieval Hint: "seatMemoryLayerTemplate memory-mining query_raw_memories detail summary agentIdentity explore_memory_history 1280 1281 20111"


## Timeline

- 2026-10-26T02:46:42Z @neo-gemini-maryam added the `enhancement` label
- 2026-10-26T02:46:43Z @neo-gemini-maryam added the `ai` label
- 2026-10-26T02:46:43Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-26T02:46:43Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-26T02:47:02Z @neo-gemini-maryam cross-referenced by #20118
- 2026-10-26T02:49:09Z @neo-gemini-maryam cross-referenced by PR #1288
- 2026-10-26T03:02:41Z @neo-gemini-maryam marked this issue as being blocked by #1280
- 2026-10-26T03:35:24Z @neo-gemini-maryam cross-referenced by PR #1285
- 2026-10-27T18:46:37Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-27T18:57:53Z @neo-gemini-maryam cross-referenced by #221
- 2026-10-27T19:00:17Z @neo-gemini-maryam cross-referenced by PR #222
- 2026-10-27T19:00:28Z @neo-gemini-maryam cross-referenced by PR #1320
- 2026-10-27T19:47:38Z @neo-gemini-maryam referenced in commit `92f5fcb` - "fix(memory-core): scope agentIdentity teaching to author-scoped sweeps (#1290)"
- 2026-10-27T20:07:14Z @tobiu referenced in commit `5710e5e` - "fix(memory-core): teach summary sweeps in seat memory and neutralize scope error (#1290) (#1320)

* fix(memory-core): teach summary sweeps in seat memory and neutralize scope error (#1290)

* fix(memory-core): scope agentIdentity teaching to author-scoped sweeps (#1290)"
- 2026-10-27T20:07:15Z @tobiu closed this issue

