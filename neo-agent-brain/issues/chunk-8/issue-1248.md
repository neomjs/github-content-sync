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
updatedAt: '2026-10-24T01:19:35Z'
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

A direct ChromaDB inspection on `2026-10-24` across the `neo_agent_memory` snapshot (`88f83b58-491f-49d1-a2a5-dc683e730ad1`, `22,571` rows) and the canonical `aiConfig.collections.memory` collection (`neo-agent-memory`, `50,517` rows) revealed that **`11,940` pre-identity rows across three founding seats (`8,132` `@neo-gemini-julia`, `1,650` `@neo-opus-ada`, `2,158` `@neo-gpt-euclid` in `neo-agent-memory`) remain unattributed** (`agentIdentity` absent) due to three structural gaps in `attributePreIdentityMemories.mjs`.

Design authority: `neomjs/neo-agent-brain#1018` (amended with `@tobiu`'s acceptance in session `7bbb249d-903b-4ad7-b69c-658b286ce4a0` and `@neo-opus-ada`'s bearer confirmation to include server-stamped `user-id` attribution plus operator-supplied era/agent bounds alongside conflict-free session propagation and journaled graph + Chroma reversibility).

## The Problem

Measured against the pre-identity rows in `neo_agent_memory` (`22,571` rows) and confirmed on the live canonical `neo-agent-memory` collection (`50,517` rows, `aiConfig.collections.memory`) on `2026-10-24`:

1. **`5,892` rows across April–June 2026 already carry a canonical agent `userId` (`2,231` `neo-gemini-julia`, `1,503` `neo-opus-ada`, `2,158` `neo-gpt-euclid`) but have no `agentIdentity`.**
   - `planAttribution()` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:222-235`) branches on `if (owner) { ... } else if (metadata.userId === SHARED_USER_ID) { ... }`.
   - Every row written after per-seat `userId` values were introduced (or renamed by `#1173`/`#1191` from `neo-gemini-pro` to `neo-gemini-julia`) but before `MemoryService` stamped `metadata.agentIdentity` is neither `owner` (`metadata.agentIdentity` is absent) nor `SHARED_USER_ID` (`'shared'`), so `planAttribution()` silently drops all `5,892` rows—and fails to propagate their identity to `shared` siblings in the same session (`52` additional rows in mixed `userId` + `shared` sessions).
   - Furthermore, `reverseAttribution()` hardcodes `userId: SHARED_USER_ID` (`line 523`), which would corrupt a `user-id`-attributed row by downgrading its original `userId` to `'shared'` on `--reverse`, and `legacyPriorOf()` (`line 394`) only accepts `SHARED_OWNER` on `memory:<uuid>` projections.

2. **`906` rows in April 2026 (`userId: 'shared'`, `agentIdentity: none`) record the seat's model/harness in `metadata.agent` (`"Antigravity (Gemini 3.1 Pro)"` `257`, `"Gemini 3.1 Pro (Antigravity)"` `100`, `"gemini"` `36`, `"Gemini 3.1 Pro"` `1`, `"Antigravity (gemini-3.1-pro)"` `1`, `"Antigravity (Gemini)"` `1`, `"Antigravity (3.1 Pro)"` `1`, `"Antigravity (Ge-3.1-Pro)"` `1`, `"Gemini 2.5 Pro"` `1`) rather than a prompt prefix.**
   - `rowEvidence()` (`line 171`) only inspects `envelopeOf(metadata?.prompt)`, never `metadata.agent`.
   - In the same window (`2026-04-10T06:26:48.150Z` onward), Claude seats (confirmed by `@tobiu` and `@neo-opus-ada` to be `@neo-opus-ada`) also wrote `shared` rows with `metadata.agent` values (`"Antigravity (Claude Opus 4.6)"`, `"claude-code"`, `"Claude Opus 4.7"`, `"Claude Opus 4.6"`, `"Claude Opus 4.6 (Antigravity)"`, `"Claude (Antigravity)"`, `"claude-opus-4-7[1m]"`) and bare `"Antigravity"` (`465` rows). An explicit `--agent <label>` / `--conflict-agent <label>` pair lets the operator attribute unambiguous model-identifying `metadata.agent` values while marking foreign `metadata.agent` values as session conflicts and leaving ambiguous `"Antigravity"`-only sessions untouched (`no-evidence`).

3. **`5,245` single-agent founding-era rows (`2025-10-02` `mem_2025-10-02T09:24:58.842Z` through `2026-04-09`, `userId: 'shared'`, `agentIdentity: none`) predate the arrival of the second peer (`2026-04-10T06:26:48.150Z`).**
   - `attributePreIdentityMemories.mjs` only matched sessions whose Turn 1 opened with `--harness "This is the Gemini CLI. We are setting up the context for our chat."`, leaving every `2025-10`–`2026-04-09` session without that literal prompt header stranded as `no-evidence` even though `@neo-gemini-julia` was the sole AI peer in the organism before `2026-04-10T00:00:00.000Z` (`GEMINI.md` founding-era ledger).

## The Architectural Reality

- `ai/scripts/migrations/attributePreIdentityMemories.mjs` is the single authority for pre-identity raw-memory attribution in both ChromaDB and the SQLite graph (`Nodes` table `AGENT_MEMORY` and legacy `MEMORY` `memory:<uuid>` projections).
- `MemoryService#queryRecentTurns` and `MemoryService#listSummaries` filter by `agentIdentity`, while `MemoryService#queryMemories` in `private` mode filters by `userId`. Rows with `userId: 'neo-gemini-julia'` (or `'neo-opus-ada'`, `'neo-gpt-euclid'`) but `agentIdentity: null` are invisible to identity-scoped turn recency and graph node queries until `agentIdentity` is stamped on both Chroma and `Nodes`.

## The Fix

Extend `ai/scripts/migrations/attributePreIdentityMemories.mjs` and `test/playwright/unit/ai/scripts/migrations/attributePreIdentityMemories.spec.mjs`:

1. **`user-id` attribution rule & `attributionPriorUserId` journal (`planAttribution`, `applyAttribution`, `reverseAttribution`, `legacyPriorOf`):**
   - Derive `userIds = new Set([...identities].map(id => id.slice(1)))`.
   - When a row has no `agentIdentity` (`!owner`):
     - If `userIds.has(metadata.userId)`: directly record `{evidence: metadata.userId, rule: 'user-id'}` without parsing `metadata.prompt` via `rowEvidence()` (since server-stamped `metadata.userId` is stronger evidence than agent-authored `[WAKE]` summary text in `prompt`), push `id` to `session.evidence`, and push the row to `session.candidates`. Session-level conflict scanning still enforces all conflicts across the session (`row of <other>`, `userId <other>`, `wake for <other>`, `agent <other>`).
     - If `metadata.userId && metadata.userId !== SHARED_USER_ID && !userIds.has(metadata.userId)`: record `session.conflicts.add(\`userId \${metadata.userId}\`)` so mixed sessions with another seat's `userId` stay untouched.
   - In `applyAttribution()`: record `attributionPriorUserId: metadata.userId` when `metadata.userId && metadata.userId !== SHARED_USER_ID` (and allow `legacyPriorOf` to move an unattributed `memory:<uuid>` projection whose `userId` matches `userIds`).
   - In `reverseAttribution()`: restore `userId: metadata.attributionPriorUserId || SHARED_USER_ID` and clear `attributionPriorUserId: null`.
2. **`--agent <label>` and `--conflict-agent <label>` CLI flags (`parseArgs`, `rowEvidence`, `planAttribution`):**
   - Add repeatable `--agent <label>` (`options.agents`) and `--conflict-agent <label>` (`options.conflictAgents`).
   - In `rowEvidence()`: if `metadata?.agent` trimmed matches `conflictAgents`, return `{conflict: \`agent \${agent}\`}`; if it matches `agents`, return `{evidence: agent, rule: 'agent-field'}`.
3. **`--before <iso-or-ms>` CLI flag (`parseArgs`, `rowEvidence`, `planAttribution`, `reportPlan`):**
   - Parse and validate `--before <value>` in `parseArgs()` (accepting ISO-8601 strings or epoch ms; throwing on invalid timestamps).
   - In `rowEvidence()`: after conflict and `--agent` checks, if `!agent` (failing closed whenever a row carries a non-empty unlisted `metadata.agent`) and `cutoffMs !== null` and `timestampMs(metadata?.timestamp) < cutoffMs`, return `{evidence: before, rule: 'before-cutoff'}`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `parseArgs(argv)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:108`) | `neomjs/neo-agent-brain#1018` + this ticket | Accepts repeatable `--agent <label>`, repeatable `--conflict-agent <label>`, and `--before <iso-or-ms>` (validated via `Date.parse` / finite number) | Throws descriptive error on missing/invalid flag value | Module JSDoc + `USAGE` | `attributePreIdentityMemories.spec.mjs` |
| `rowEvidence(metadata, rules)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:218`) | `neomjs/neo-agent-brain#1018` + this ticket | Evaluates conflict gates (`wake-addressee` to foreign seat, `--conflict-agent` match) before positive evidence (`harness-marker`, `wake-addressee`, `agent-field`, and fail-closed `before-cutoff` when `!agent`) | Returns `null` when no rule or conflict matches | Function JSDoc | `attributePreIdentityMemories.spec.mjs` |
| `planAttribution(options)` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:272`) | `neomjs/neo-agent-brain#1018` + this ticket | Attributes `!owner` rows whose `metadata.userId` is in `userIds` (`rule: 'user-id'`), propagates across conflict-free sessions, and records `userId <other>` conflicts for foreign `userId` rows | Leaves conflict or no-evidence sessions untouched | Function JSDoc | `attributePreIdentityMemories.spec.mjs` + live dry-run receipts |
| `applyAttribution` / `reverseAttribution` (`ai/scripts/migrations/attributePreIdentityMemories.mjs:516,585`) | `neomjs/neo-agent-brain#1018` + this ticket | Journals `attributionPriorUserId` when re-keying a non-`shared` row and restores `metadata.attributionPriorUserId \|\| 'shared'` on `--reverse` | Defaults to `'shared'` when `attributionPriorUserId` is absent | Function JSDoc | `attributePreIdentityMemories.spec.mjs` |

## Decision Record impact

Amends `neomjs/neo-agent-brain#1018`'s evidence-boundary scope (recorded on `#1018` with `@tobiu`'s acceptance in session `7bbb249d-903b-4ad7-b69c-658b286ce4a0` and `@neo-opus-ada`'s bearer confirmation) to include server-stamped `user-id` attribution plus operator-supplied era (`--before`, failing closed on any non-empty unlisted `metadata.agent`) and model-label (`--agent` / `--conflict-agent`) bounds.

## Acceptance Criteria

- [ ] AC-1 `planAttribution` attributes rows lacking `agentIdentity` whose `metadata.userId` matches `--identity` or `--alias` (`rule: 'user-id'`), propagates to `shared` siblings in the same session, marks foreign `userId` rows as session conflicts, and `reverseAttribution` restores `attributionPriorUserId`.
- [ ] AC-2 `parseArgs`, `rowEvidence`, and `planAttribution` support repeatable `--agent <label>` (`rule: 'agent-field'`), repeatable `--conflict-agent <label>` (`conflict: 'agent <label>'`), and `--before <iso-or-ms>` (`rule: 'before-cutoff'`, failing closed on non-empty unlisted `metadata.agent`).
- [ ] AC-3 Unit tests in `test/playwright/unit/ai/scripts/migrations/attributePreIdentityMemories.spec.mjs` cover `user-id` attribution + reversal, `--agent` / `--conflict-agent` conflict resolution, and `--before` cutoff boundary + fail-closed unlisted `metadata.agent` + invalid input handling, and pass completely.
- [ ] AC-4 Read-only dry-run (`mode: 'DRY-RUN'`) against the live ChromaDB `neo-agent-memory` collection (`aiConfig.collections.memory`, `50,517` rows) succeeds and reports the planned counts and per-rule sample for `@neo-gemini-julia`, `@neo-opus-ada`, and `@neo-gpt-euclid`.
- [ ] AC-5 **Post-merge:** Operator-approved `--apply --offline` execution with Memory Core graph writers stopped (tracked alongside `#1149` / `#1018`).

## Out of Scope

- Running `--apply --offline` while live Memory Core containers/daemons hold open SQLite graph connections (post-merge offline step).
- Attributing the `436` April 2026 `"Antigravity"`-only `shared` rows where `metadata.agent` does not name a model family and no session marker/wake exists.

## Avoided Traps

- **Restoring `userId: 'shared'` unconditionally in `reverseAttribution()`:** Rows attributed via `rule: 'user-id'` already carried `userId: 'neo-gemini-julia'` (or `'neo-opus-ada'`, `'neo-gpt-euclid'`) before the migration. Overwriting `userId` with `'shared'` on `--reverse` would destroy their pre-existing `private`-recall ownership. Journaling `attributionPriorUserId` preserves exact reversibility.
- **Treating bare `metadata.agent === "Antigravity"` as single-seat evidence in April 2026:** Both Gemini 3.1 Pro and Claude Opus 4.6 used Antigravity in mid-April 2026 (`"Antigravity (Claude Opus 4.6)"`). Only model-specific `metadata.agent` strings are passed via `--agent`, while foreign family strings are passed via `--conflict-agent` and bare `"Antigravity"` sessions without other evidence remain `shared`.

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
- 2026-10-24T01:08:19Z @neo-gemini-julia referenced in commit `7355476` - "feat(migrations): clarify --before vs --agent rule independence in attributePreIdentityMemories (#1248)"
- 2026-10-24T01:19:35Z @neo-gemini-julia referenced in commit `f25e270` - "feat(migrations): fail closed on unlisted metadata.agent in before-cutoff rule (#1248)"

