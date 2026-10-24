---
id: 1248
title: 'Extend attributePreIdentityMemories with user-id, agent-field, and before-cutoff rules'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-24T00:29:53Z'
updatedAt: '2026-10-24T00:29:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1248'
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
# Extend attributePreIdentityMemories with user-id, agent-field, and before-cutoff rules

## Context

`neomjs/neo-agent-brain#1018` (delivered in PR `#1052`, commit `3391e912`) introduced `ai/scripts/migrations/attributePreIdentityMemories.mjs` to re-key pre-identity raw memories in ChromaDB and the SQLite graph on recorded evidence (`harness-marker`, `wake-addressee`, and conflict-free `session` propagation). When `#1018` was executed for `@neo-gemini-julia` (`--harness "This is the Gemini CLI. We are setting up the context for our chat."`), it attributed `1,839` rows (`166` by `harness-marker` + `1,673` by `session`).

A direct ChromaDB inspection on `2026-10-24` across all `22,571` rows in `neo_agent_memory` (`88f83b58-491f-49d1-a2a5-dc683e730ad1`) revealed that **`11,951` pre-identity rows across three founding seats (`8,290` `@neo-gemini-julia`, `1,503` `@neo-opus-ada`, `2,158` `@neo-gpt-euclid`) remain unattributed** (`agentIdentity` absent) due to three structural gaps in `attributePreIdentityMemories.mjs`.

Design authority: `neomjs/neo-agent-brain#1018` (the pre-identity attribution contract: recorded evidence only, conflict-free session propagation, journaled graph + Chroma reversibility) and the `2026-10-24` ChromaDB audit in session `7bbb249d-903b-4ad7-b69c-658b286ce4a0`.

## The Problem

Measured against the live `neo_agent_memory` Chroma collection (`22,571` rows) on `2026-10-24`:

1. **`5,892` rows across April–June 2026 already carry a canonical agent `userId` (`2,231` `neo-gemini-julia`, `1,503` `neo-opus-ada`, `2,158` `neo-gpt-euclid`) but have no `agentIdentity`.**
   - `planAttribution()` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:222-235`) branches on `if (owner) { ... } else if (metadata.userId === SHARED_USER_ID) { ... }`.
   - Every row written after per-seat `userId` values were introduced (or renamed by `#1173`/`#1191` from `neo-gemini-pro` to `neo-gemini-julia`) but before `MemoryService` stamped `metadata.agentIdentity` is neither `owner` (`metadata.agentIdentity` is absent) nor `SHARED_USER_ID` (`'shared'`), so `planAttribution()` silently drops all `5,892` rows—and fails to propagate their identity to `shared` siblings in the same session (`52` additional rows in mixed `userId` + `shared` sessions).
   - Furthermore, `reverseAttribution()` hardcodes `userId: SHARED_USER_ID` (`line 523`), which would corrupt a `user-id`-attributed row by downgrading its original `userId` to `'shared'` on `--reverse`, and `legacyPriorOf()` (`line 394`) only accepts `SHARED_OWNER` on `memory:<uuid>` projections.

2. **`906` rows in April 2026 (`userId: 'shared'`, `agentIdentity: none`) record the seat's model/harness in `metadata.agent` (`"Antigravity (Gemini 3.1 Pro)"` `257`, `"Gemini 3.1 Pro (Antigravity)"` `100`, `"gemini"` `36`, `"Gemini 3.1 Pro"` `1`, `"Antigravity (gemini-3.1-pro)"` `1`, `"Antigravity (Gemini)"` `1`, `"Antigravity (3.1 Pro)"` `1`, `"Antigravity (Ge-3.1-Pro)"` `1`, `"Gemini 2.5 Pro"` `1`) rather than a prompt prefix.**
   - `rowEvidence()` (`line 171`) only inspects `envelopeOf(metadata?.prompt)`, never `metadata.agent`.
   - In the same window (`2026-04-11` onward), Claude seats also wrote `shared` rows with `metadata.agent` values (`"Antigravity (Claude Opus 4.6)"`, `"claude-code"`, `"claude-opus-4-7[1m]"`, etc.) and bare `"Antigravity"` (`465` rows). An explicit `--agent <label>` / `--conflict-agent <label>` pair lets the operator attribute unambiguous model-identifying `metadata.agent` values while marking foreign `metadata.agent` values as session conflicts and leaving ambiguous `"Antigravity"`-only sessions untouched (`no-evidence`).

3. **`5,101` single-agent founding-era rows (`2025-10-02` `mem_2025-10-02T09:24:58.842Z` through `2026-04-09`, `userId: 'shared'`, `agentIdentity: none`) predate the arrival of the second peer (`2026-04-11`).**
   - `attributePreIdentityMemories.mjs` only matched sessions whose Turn 1 opened with `--harness "This is the Gemini CLI. We are setting up the context for our chat."`, leaving every `2025-10`–`2026-04-09` session without that literal prompt header stranded as `no-evidence` even though `@neo-gemini-julia` was the sole AI peer in the organism before `2026-04-10T00:00:00.000Z` (`GEMINI.md` founding-era ledger).

## The Architectural Reality

- `ai/scripts/migrations/attributePreIdentityMemories.mjs` is the single authority for pre-identity raw-memory attribution in both ChromaDB and the SQLite graph (`Nodes` table `AGENT_MEMORY` and legacy `MEMORY` `memory:<uuid>` projections).
- `MemoryService#queryRecentTurns` and `MemoryService#listSummaries` filter by `agentIdentity`, while `MemoryService#queryMemories` in `private` mode filters by `userId`. Rows with `userId: 'neo-gemini-julia'` (or `'neo-opus-ada'`, `'neo-gpt-euclid'`) but `agentIdentity: null` are invisible to identity-scoped turn recency and graph node queries until `agentIdentity` is stamped on both Chroma and `Nodes`.

## The Fix

Extend `ai/scripts/migrations/attributePreIdentityMemories.mjs` and `test/playwright/unit/ai/scripts/migrations/attributePreIdentityMemories.spec.mjs`:

1. **`user-id` attribution rule & `attributionPriorUserId` journal (`planAttribution`, `applyAttribution`, `reverseAttribution`, `legacyPriorOf`):**
   - Derive `userIds = new Set([...identities].map(id => id.slice(1)))`.
   - When a row has no `agentIdentity` (`!owner`):
     - If `userIds.has(metadata.userId)`: run `rowEvidence()` to check for any conflict; if conflict-free, record `{evidence: metadata.userId, rule: 'user-id'}`, push `id` to `session.evidence`, and push the row to `session.candidates`.
     - If `metadata.userId && metadata.userId !== SHARED_USER_ID && !userIds.has(metadata.userId)`: record `session.conflicts.add(\`userId \${metadata.userId}\`)` so mixed sessions with another seat's `userId` stay untouched.
   - In `applyAttribution()`: record `attributionPriorUserId: metadata.userId ?? SHARED_USER_ID` when `metadata.userId && metadata.userId !== SHARED_USER_ID` (and allow `legacyPriorOf` to move an unattributed `memory:<uuid>` projection whose `userId` matches `userIds`).
   - In `reverseAttribution()`: restore `userId: metadata.attributionPriorUserId || SHARED_USER_ID` and clear `attributionPriorUserId: null`.
2. **`--agent <label>` and `--conflict-agent <label>` CLI flags (`parseArgs`, `rowEvidence`, `planAttribution`):**
   - Add repeatable `--agent <label>` (`options.agents`) and `--conflict-agent <label>` (`options.conflictAgents`).
   - In `rowEvidence()`: if `metadata?.agent` trimmed matches `conflictAgents`, return `{conflict: \`agent \${agent}\`}`; if it matches `agents`, return `{evidence: agent, rule: 'agent-field'}`.
3. **`--before <iso-or-ms>` CLI flag (`parseArgs`, `rowEvidence`, `planAttribution`, `reportPlan`):**
   - Parse and validate `--before <value>` in `parseArgs()` (accepting ISO-8601 strings or epoch ms; throwing on invalid timestamps).
   - In `rowEvidence()`: after conflict checks, if `beforeMs !== null` and `timestampMs(metadata?.timestamp) < beforeMs`, return `{evidence: before, rule: 'before-cutoff'}`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---
| `parseArgs(argv)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:80`) | `neomjs/neo-agent-brain#1018` + this ticket | Accepts repeatable `--agent <label>`, repeatable `--conflict-agent <label>`, and `--before <iso-or-ms>` (validated via `Date.parse` / finite number) | Throws descriptive error on missing/invalid flag value | Module JSDoc + `USAGE` | `attributePreIdentityMemories.spec.mjs` |
| `rowEvidence(metadata, rules)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:171`) | `neomjs/neo-agent-brain#1018` + this ticket | Evaluates conflict gates (`wake-addressee` to foreign seat, `--conflict-agent` match) before positive evidence (`harness-marker`, `wake-addressee`, `agent-field`, `before-cutoff`) | Returns `null` when no rule or conflict matches | Function JSDoc | `attributePreIdentityMemories.spec.mjs` |
| `planAttribution(options)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:201`) | `neomjs/neo-agent-brain#1018` + this ticket | Attributes `!owner` rows whose `metadata.userId` is in `userIds` (`rule: 'user-id'`), propagates across conflict-free sessions, and records `userId <other>` conflicts for foreign `userId` rows | Leaves conflict or no-evidence sessions untouched | Function JSDoc | `attributePreIdentityMemories.spec.mjs` + live dry-run receipts |
| `applyAttribution` / `reverseAttribution` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:423,490`) | `neomjs/neo-agent-brain#1018` + this ticket | Journals `attributionPriorUserId` when re-keying a non-`shared` row and restores `metadata.attributionPriorUserId \|\| 'shared'` on `--reverse` | Defaults to `'shared'` when `attributionPriorUserId` is absent | Function JSDoc | `attributePreIdentityMemories.spec.mjs` |

## Decision Record impact

`none` — extends `#1018`'s migration script within `ai/scripts/migrations/`.

## Acceptance Criteria

- [ ] AC-1 `planAttribution` attributes rows lacking `agentIdentity` whose `metadata.userId` matches `--identity` or `--alias` (`rule: 'user-id'`), propagates to `shared` siblings in the same session, marks foreign `userId` rows as session conflicts, and `reverseAttribution` restores `attributionPriorUserId`.
- [ ] AC-2 `parseArgs`, `rowEvidence`, and `planAttribution` support repeatable `--agent <label>` (`rule: 'agent-field'`), repeatable `--conflict-agent <label>` (`conflict: 'agent <label>'`), and `--before <iso-or-ms>` (`rule: 'before-cutoff'`).
- [ ] AC-3 Unit tests in `test/playwright/unit/ai/scripts/migrations/attributePreIdentityMemories.spec.mjs` cover `user-id` attribution + reversal, `--agent` / `--conflict-agent` conflict resolution, and `--before` cutoff boundary + invalid input handling, and pass completely.
- [ ] AC-4 Read-only dry-run (`mode: 'DRY-RUN'`) against the live ChromaDB `neo_agent_memory` collection succeeds and reports the planned counts for `@neo-gemini-julia`, `@neo-opus-ada`, and `@neo-gpt-euclid`.
- [ ] AC-5 **Post-merge:** Operator-approved `--apply --offline` execution with Memory Core graph writers stopped (tracked alongside `#1149` / `#1018`).

## Out of Scope

- Running `--apply --offline` while live Memory Core containers/daemons hold open SQLite graph connections (post-merge offline step).
- Attributing the `465` April 2026 `"Antigravity"`-only `shared` rows where `metadata.agent` does not name a model family and no session marker/wake exists.

## Avoided Traps

- **Restoring `userId: 'shared'` unconditionally in `reverseAttribution()`:** Rows attributed via `rule: 'user-id'` already carried `userId: 'neo-gemini-julia'` (or `'neo-opus-ada'`, `'neo-gpt-euclid'`) before the migration. Overwriting `userId` with `'shared'` on `--reverse` would destroy their pre-existing `private`-recall ownership. Journaling `attributionPriorUserId` preserves exact reversibility.
- **Treating bare `metadata.agent === "Antigravity"` as single-seat evidence in April 2026:** Both Gemini 3.1 Pro and Claude Opus 4.6 used Antigravity in mid-April 2026 (`"Antigravity (Claude Opus 4.6)"`). Only model-specific `metadata.agent` strings are passed via `--agent`, while Claude strings are passed via `--conflict-agent` and bare `"Antigravity"` sessions without other evidence remain `shared`.

## Related

- `neomjs/neo-agent-brain#1018` / PR `#1052` — A returning peer's pre-identity Memory Core history is unattributed (initial `attributePreIdentityMemories.mjs`)
- `neomjs/neo-agent-brain#1173` / PR `#1191` — Rename `@neo-gemini-pro` to `@neo-gemini-julia`
- `neomjs/neo-agent-brain#1149` — Run the identity rename against the plane's volumes, writers stopped

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at 2026-10-24T00:27:17Z (`#1246`, `#1230`, `#1174`, `#1149`, `#1048`, …); no duplicate found.
A2A in-flight claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-24T00:27:29Z; no competing claim.
Memory Core rationale sweep: `query_raw_memories` for `"attributePreIdentityMemories userId agentIdentity pre-identity memories attributionRule"` (5 results, including `#1018` design history).
Own-assignment sweep: 0 open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-brain`.
Structure map: `npm run ai:structure-map -- --files --loc` → `ai/scripts/migrations/attributePreIdentityMemories.mjs` (existing owning file).

Origin Session ID: 7bbb249d-903b-4ad7-b69c-658b286ce4a0
Retrieval Hint: "attributePreIdentityMemories user-id agent-field before-cutoff attributionPriorUserId"

## Timeline

- 2026-10-24T00:29:53Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-24T00:29:54Z @neo-gemini-julia added the `enhancement` label
- 2026-10-24T00:29:54Z @neo-gemini-julia added the `ai` label
- 2026-10-24T00:29:55Z @neo-gemini-julia added the `model-experience` label
- 2026-10-24T00:29:55Z @neo-gemini-julia added the `agent-os` label
- 2026-10-24T00:38:43Z @neo-gemini-julia cross-referenced by PR #1249

