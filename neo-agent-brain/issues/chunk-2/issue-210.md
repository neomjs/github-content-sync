---
id: 210
title: Session-scoped memory reads ship the per-turn miniSummary
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-28T21:12:52Z'
updatedAt: '2026-08-30T12:24:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/210'
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
closedAt: '2026-08-30T12:24:53Z'
---
# Session-scoped memory reads ship the per-turn miniSummary

## Context

Operator direction on the Institution memories-pane design (neo-agent-institution#20, 2026-08-28): raw turn prose cannot be a row headline — "a title could be the tweet size mini summary. real responses can be VERY long." The measured facts from the design session confirm it, and the field already exists — only the session-scoped read fails to ship it.

## The Problem

The per-turn `miniSummary` is generated (the `backfillMiniSummaries` pipeline, `mc-mini-summary` embedding stage) and stored on the graph row (`properties.miniSummary`). `queryRecentTurns` joins and returns it (`MemoryService.mjs:1693` `json_extract(memory.data, '$.properties.miniSummary')`; `detail:'summary'` path). But the session-scoped read — `listMemories({sessionId, limit, offset, memorySharing})`, exposed as `get_session_memories` and passed through unchanged by the Fleet session-memories source — returns only `{id, sessionId, timestamp, prompt, thought, response, type, agent, model, amountToolCalls, toolsUsed}`. A session drill that wants tweet-size turn titles has no path to them.

Measured on a live design session (11 turns, 2026-08-28): thoughts run 900–2,100 chars, responses 150–260 chars only under one agent's discipline (the contract has no upper bound); backfilled miniSummaries measure 120–250 chars and read as genuine turn titles.

## The Architectural Reality

- `MemoryService.listMemories()` reads Chroma for the prose fields; the miniSummary lives on the SQLite graph row. The join pattern already exists in `queryRecentTurns` (same service, `:1693`).
- The truncated-raw fallback helper (used when no miniSummary exists yet — backfill is scheduled, not inline; `:1261`) is shared service substrate and applies identically here, with the existing `summaryFallback` marker so consumers can tell a real mini-summary from a truncation.
- The Fleet session-memories source passes MC rows through unchanged — one added field reaches the Institution drill with zero adapter logic.

## The Fix

Extend `listMemories()` to join the graph-side `miniSummary` per returned row (reusing the `queryRecentTurns` extraction), returning `miniSummary` plus the existing `summaryFallback` semantics (fallback = truncated raw, marked). `get_session_memories` OpenAPI response schema gains the field; the Fleet pass-through ships it automatically. No new detail parameter on this operation (per the #37 review ruling: `detail` belongs to `queryRecentTurns`; the session read stays full — this ticket ADDS a field, it does not add a mode).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `listMemories()` return rows | graph row `properties.miniSummary` (exists; `:1693` pattern) | each row carries `miniSummary` + `summaryFallback` | truncated-raw fallback, marked (existing helper) | service JSDoc + OpenAPI schema | unit: backfilled row ships real summary; fresh row ships marked fallback |
| `get_session_memories` response schema | OpenAPI | field added, additive only | absent field never breaks old consumers | openapi.yaml | schema test |
| Fleet session-memories pass-through | unchanged-forwarding contract | field flows through with zero adapter change | n/a | existing source doc | consumer read shows the field |

## Decision Record impact

None — additive field on an existing read, reusing existing storage and join patterns.

## Acceptance Criteria

- [ ] `listMemories()` rows carry `miniSummary` (graph-joined) and `summaryFallback` (true only for the truncation fallback).
- [ ] `get_session_memories` OpenAPI schema documents both fields; additive, no breaking change.
- [ ] The Fleet session-memories source forwards them unchanged (verified by a consumer read).
- [ ] No `detail` parameter is added to the session-scoped operation.

## Out of Scope

- New title fields or a second summarization pipeline (the miniSummary IS the turn title — inventing a sibling field is duplicate substrate).
- Session-level titles (the Dream summary's `title` already serves; live sessions surface through `query_recent_turns`).
- The Institution drill UI consuming this (neo-agent-institution#20's implementation subs).

## Related

Consumer: neo-agent-institution#20 (memories pane — the drill's turn-title row) · neo-agent-institution PR #38 (the design that names this dependency).

Live latest-open sweep: checked latest 20 open issues at 2026-08-28T21:20Z — no equivalent. A2A in-flight claim sweep: recent window scanned, no overlapping lane-claim.

Origin Session ID: 4f07d934-f43f-4406-98a0-9562e122e471

Retrieval Hint: `query_raw_memories("session drill miniSummary turn title tweet size")`

Authored by Clio (Fable 5, Claude Code). Session 41859592-b7ee-4bce-bee3-f25644d9003b.


## Timeline

- 2026-08-28T21:12:54Z @neo-fable-clio added the `enhancement` label
- 2026-08-28T21:12:54Z @neo-fable-clio added the `ai` label
- 2026-08-28T21:12:54Z @neo-fable-clio added the `architecture` label
- 2026-08-28T21:20:37Z @neo-fable-clio cross-referenced by PR #38
- 2026-08-30T01:13:23Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-30T01:30:11Z @neo-gpt-emmy referenced in commit `8bde305` - "test(memory-core): pin session summary schema (#210)"
- 2026-08-30T01:31:38Z @neo-gpt-emmy cross-referenced by PR #248
- 2026-08-30T12:24:53Z @tobiu referenced in commit `91cb280` - "Merge pull request #248 from neomjs/codex/210-session-mini-summary

feat(memory-core): expose session mini summaries (#210)"
- 2026-08-30T12:24:53Z @tobiu closed this issue

