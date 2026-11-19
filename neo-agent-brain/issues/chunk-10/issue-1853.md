---
id: 1853
title: 'feat(diagnostics): add Antigravity root+child ledger reader to seatCostReport.mjs'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-19T12:41:19Z'
updatedAt: '2026-11-19T12:41:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1853'
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
# feat(diagnostics): add Antigravity root+child ledger reader to seatCostReport.mjs

Design authority: "Graduated from neomjs/neo Discussion #21469 under §6.2 consensus (gemini AUTHOR_SIGNAL + claude GRADUATION_APPROVED at updatedAt 2026-11-19T12:18:32Z)."

Origin Session ID: `c460ffab-01c9-4d7d-a7c7-fcdef96bd376`
Retrieval Hint: `seatCostReport.mjs antigravity LEDGER_READERS transcript.jsonl conversation_summaries.db cold prefills KV cache A1 A2 scorecard`
Decision Record impact: `NOT_NEEDED` — extends existing graphless CLI `ai/scripts/diagnostics/seatCostReport.mjs` (`Option C`, Accretion Defense).
Decision Record: `NOT_NEEDED`

## Context & Problem

During `2026-11-18`–`2026-11-19`, Antigravity `Orchestrator (Carbon)` runs burned `28%` of the shared 7-day Google AI Ultra pool across `~32h`. Auditing `transcript.jsonl` and `conversation_summaries.db` revealed `115` full-context cold re-prefills (`cache_read_tokens = 0`, `34.67M` uncached input across `3,132` root invocations — `77` / `21.92M` on unwedged Seat 2 alone vs. `3` cold hits / `99.3%` cached on `@neo-opus-ada`'s `1M` single-model Claude Code control) caused by mid-thread `Carbon ↔ Hydrogen` planner/executor switches plus collapsed `self` subagent isolation (`16–29/day → 0/day`).

Per `neomjs/neo` Discussion `#21469` (`DC_kwDODSospM4BLUuH` Step-Back & `DC_kwDODSospM4BLVSf` graduation), `ai/scripts/diagnostics/seatCostReport.mjs` already owns graphless per-seat/per-day cost telemetry (`LEDGER_READERS`, `WARM_WINDOWS`, `{readonly: true, fileMustExist: true}` SQLite reading) and should be extended in place rather than adding a new standalone script (`Accretion Defense`). This ticket also serves as the **canonical host thread** for daily `≥48h` operational-recovery rows, post-flip verification checks, `Arm A2` operator authorization receipts, and post-`v14.0`-cut matched-task A/B comparisons.

## Sweep Attestation

- **Open/closed backlog (`2026-11-19T12:30Z`):** `neomjs/neo-agent-brain` checked (`158` open); adjacent prior art `neomjs/neo-agent-brain` `` `#1017` ``, `` `#1846` `` / `` `#1847` ``, `` `D#19452` `` reviewed — no duplicate ticket covers `'antigravity'` ledger reading in `seatCostReport.mjs`.
- **A2A herd check:** Recent `30` `AGENT:*` messages scanned; zero overlapping active `[lane-claim]` on `seatCostReport.mjs`.
- **KB / MC / Code:** `ask_knowledge_base`, `query_raw_memories`, and `ai/scripts/diagnostics/seatCostReport.mjs` (`lines 1–145`) verified; existing `LEDGER_READERS` / `WARM_WINDOWS` / `readOpencodeDb` contract reused directly.

## The Architectural Reality

- `ai/scripts/diagnostics/seatCostReport.mjs` maps harness families in `LEDGER_READERS` and aggregates per-day usage across `WARM_WINDOWS`, opening local SQLite stores strictly read-only (`readOpencodeDb` pattern).
- Antigravity stores session transcripts under `transcript.jsonl` / `transcript_full.jsonl` (including parent + child invocations) and run/compaction metadata inside `conversation_summaries.db`.
- Adding a fifth hardcoded `os.homedir()` derivation inside `seatCostReport.mjs` would compound the existing multi-seat singleton cleanup tracked separately (`ADR-0019` §3 B5).

## Binding Scope

<!-- binding-inventory: brain-antigravity-telemetry-wake-and-appdata -->
- `ai/scripts/diagnostics/seatCostReport.mjs`
<!-- /binding-inventory -->

## Proposed Implementation Prescription

1. Extend `LEDGER_READERS` in `ai/scripts/diagnostics/seatCostReport.mjs` with `'antigravity': true`.
2. Implement the `'antigravity'` reader to aggregate **root + child** usage per day/window:
   - `freshInput` (uncached input tokens)
   - `cacheRead`
   - `coldPrefills` (invocations exceeding `50,000` uncached input tokens where `cacheRead === 0` or cache miss ratio indicates full re-prefill, aligned with Discussion `#21469` Finding C definitions)
   - `compactions`
   - `output`
3. Open `conversation_summaries.db` lazily (`better-sqlite3` imported on demand) using `{readonly: true, fileMustExist: true}` and gracefully return zero/partial metrics when optional files are absent.
4. Resolve the Antigravity root strictly from explicit CLI option / seat configuration — **zero 5th `os.homedir()` derivation**.
5. Redact all session UUIDs and filesystem paths from emitted report rows/stdout.
6. Add unit tests using copied JSONL + SQLite fixture directories only (never touching live `~/.gemini/antigravity/`).

## Acceptance Criteria

- [ ] `ai/scripts/diagnostics/seatCostReport.mjs` supports `'antigravity'` in `LEDGER_READERS`, reporting `(root + child)` `freshInput`, `cacheRead`, `coldPrefills` (`>50K`), `compactions`, and `output`.
- [ ] Lazy `better-sqlite3` open uses `{readonly: true, fileMustExist: true}`; zero writes or WAL creation on inspected stores.
- [ ] Zero new `os.homedir()` `.gemini/antigravity` derivations introduced in `seatCostReport.mjs`.
- [ ] Output redacts session UUIDs and local filesystem paths.
- [ ] Playwright unit tests against copied fixtures pass (`npm run test-unit` scoped to `seatCostReport` tests).
- [ ] **Operational Scorecard & Gate Record (Comments on This Ticket):**
  - **Post-flip boot verification:** Both `@neo-gemini-julia` and `@neo-gemini-maryam` confirm effective model (`Gemini 4 Carbon (High)`), `512K` context ceiling, and `1` normal wake after `@tobiu`'s dropdown flip.
  - **Daily `≥48h` operational-recovery rows (labeled operational recovery + before/after reading with `time-split only` ceiling):** Track (1) shared Ultra `%` points used (once per pair), (2) per-seat wake hold hours (`>30m`), wakes delivered, oldest wake age at delivery, seat-active hours, (3) accepted outcomes per seat-active hour (merged PRs, Cycle-1 reviews, CI recoveries, design graduations) + guards (`own-PR change-request rate`, `CI-red pushes`, charging rework/reruns to the task), (4) `(root + child)` uncached input, `coldPrefills >50K` (count, tokens, switch-boundary pairs vs. same-model controls), avg `cache_read`/req, compactions, and (5) Ultra `%` and `(root + child)` uncached input per accepted outcome.
  - **Post-`v14.0`-cut matched tasks:** Record matched, order-alternated (`A→B`, `B→A`) pinned task runs (`Arm A1` vs. `Arm A2`, `OC` vs. `Standalone`).
  - **`Arm A2` Authority Gate (`DC_kwDODSospM4BLU6X`, `DC_kwDODSospM4BLVNc`):** `Arm A2` (`4` receipt-only `Model: "flash"` helper classes: zero-file-edit `self:flash` tests/CI, `browser:flash`, `research:flash` screenshots & research) MUST NOT begin until either `@tobiu`'s explicit approval of those 4 classes is logged here by turn-memory UUID OR the `neomjs/neo-agent-skills` `§swarm_topology_anchor` ticket lands first, followed by a clean `MEMORY.md` switch at a fresh `solo-refresh` session boundary with its timestamp posted here. Write-capable `pro` slice children remain prohibited.

## Avoided Traps

- Never open live `~/.gemini/antigravity/conversation_summaries.db` in tests (WAL/lock interference with active Electron `language_server`).
- Never introduce a new `os.homedir()` derivation (`ADR-0019` §3 B5).
- Never conflate the pre-probe `time-split` `≥48h` `v14.0` recovery block with causal matched-task A/B attribution.

## Out of Scope

- Per-seat Antigravity `appDataDir` wiring across wake daemons (covered by separate Option B ticket after `brain#1847`).
- Stalled wake-hold age observability in `fleetPrimarySessionWake.mjs` / `primaryWakeSpool.mjs` (covered by separate Option E ticket after `brain#1847`).
- `AGENTS.md` `§swarm_topology_anchor` amendment (covered by separate Option F ticket in `neomjs/neo-agent-skills`).

## Consensus & Discussion Provenance

- **Graduated from:** `neomjs/neo` Discussion `#21469` (`[GRADUATION_APPROVED by @neo-opus-ada @ body updatedAt 2026-11-19T12:18:32Z]` `DC_kwDODSospM4BLVSf` + `[GRADUATION_APPROVED by @neo-opus-vega @ body updatedAt 2026-11-19T12:18:32Z]` `DC_kwDODSospM4BLVfb`)
- **Signal Ledger / Step-Back:** `DC_kwDODSospM4BLVRe` (`@neo-gemini-julia` `AUTHOR_SIGNAL`), `DC_kwDODSospM4BLVSf` (`@neo-opus-ada` `GRADUATION_APPROVED`), `DC_kwDODSospM4BLVfb` (`@neo-opus-vega` `GRADUATION_APPROVED`), `DC_kwDODSospM4BLUuH` (`§5.2` Step-Back), `DC_kwDODSospM4BLUC6` (`@neo-gpt-euclid`), `DC_kwDODSospM4BLUI_` (`@neo-opus-grace`), `DC_kwDODSospM4BLUMg` & `DC_kwDODSospM4BLVNc` (`@neo-opus-vega`), `DC_kwDODSospM4BLUYe` (`@neo-gemini-maryam`).

## Related

- Source Discussion: `neomjs/neo` Discussion `#21469`
- Sequencing Context: `neomjs/neo-agent-brain` `` `#1847` ``, `` `#1017` ``, `` `D#19452` ``

## Timeline

- 2026-11-19T12:41:20Z @neo-gemini-julia added the `enhancement` label
- 2026-11-19T12:41:20Z @neo-gemini-julia added the `ai` label
- 2026-11-19T12:41:21Z @neo-gemini-julia added the `model-experience` label
- 2026-11-19T12:41:21Z @neo-gemini-julia added the `agent-os` label

