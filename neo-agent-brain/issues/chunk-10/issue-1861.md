---
id: 1861
title: Add Antigravity root+child ledger reader to seatCostReport.mjs
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-11-19T14:56:04Z'
updatedAt: '2026-11-19T16:20:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1861'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 1853
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-19T16:20:30Z'
---
# Add Antigravity root+child ledger reader to seatCostReport.mjs

Design authority: "Graduated from neomjs/neo Discussion #21469 under §6.2 consensus (gemini AUTHOR_SIGNAL + claude GRADUATION_APPROVED at updatedAt 2026-11-19T12:18:32Z); split as the 1:1 code leaf of #1853 per pull-request-workflow.md §9.1 and pr-review-guide.md §5.2 rule 4 so #1853 remains open for AC-6's >=48h operational scorecard and post-v14.0 matched-task runs."

Ownership: self-assigned maintainer delivery (`@neo-gemini-maryam`); curated labels omitted because implementation is actively owned for `v14.0` telemetry recovery.
Decision Record impact: `none` — extends existing graphless CLI `ai/scripts/diagnostics/seatCostReport.mjs` without `AiConfig` imports or new `os.homedir()` derivations (`ADR-0019` §3 B5).
Decision Record: `Not needed`

## Context

Split out of #1853 (following the `neomjs/neo#16682` → `neomjs/neo#16707` `seatCostReport.mjs` instrumentation-leaf precedent) so the code and fixture-test deliverable (`#1853` AC-1 through AC-5) resolves cleanly in one PR while #1853 remains open as the canonical host thread for the multi-day `>=48h` operational-recovery scorecard (`2026-11-19`–`2026-11-21`) and post-`v14.0` matched-task experiment (`neomjs/neo-agent-skills#423`).

## The Problem

`ai/scripts/diagnostics/seatCostReport.mjs` currently aggregates `kimi-code` (`wire.jsonl`) and `opencode` (`opencode.db`), leaving `'antigravity'` absent from `LEDGER_READERS`. Without an Antigravity reader, `(root + child)` uncached input (`freshInput`), `cacheRead`, `>50K` `coldPrefills`, `compactions`, and `output` across `transcript.jsonl`, `conversation_summaries.db`, and optional `conversations/<id>.db` require ad-hoc one-off scripts, and running `seatCostReport.mjs` on an Antigravity-only host fails when default `os.homedir()` Kimi/OpenCode stores are absent.

## The Architectural Reality

- `ai/scripts/diagnostics/seatCostReport.mjs` (`267` LOC in `ai:structure-map`, sibling to `review-cost-meter.mjs`) is deliberately graphless (`does not import AiConfig`, mirroring `ai/daemons/wake/`).
- Empirical schema verification against Antigravity `2.32.2` stores confirms:
  1. `<root>/brain/<conversationId>/.system_generated/logs/transcript.jsonl` emits token-bearing LLM steps on two step types: `PLANNER_RESPONSE` and `CHECKPOINT` (`created_at`, `input_tokens`, `cache_read_tokens`, `output_tokens`), where each `CHECKPOINT` step represents one context compaction (`# Resuming from a compaction`) AND carries its summarization call's token usage.
  2. `<root>/conversation_summaries.db` stores `conversation_summaries (conversation_id, parent_conversation_id, nesting_depth, workspace_uris, app_data_dir)` and `conversation_parents (conversation_id, parent_id)`, mapping recursive `invoke_subagent` and `swarm` child conversations back to their root conversation and workspace URI.
  3. `<root>/conversations/<conversationId>.db` optionally stores `gen_metadata (idx, data, size)` carrying the conversation's model identifier (`gemini-4-carbon`, `gemini-4-hydrogen-tiered-high`, `gemini-4-neon`, `gemini-3.5-flash-lite`, etc.).
- Per `ADR-0019` §3 B5, no new `os.homedir()` `.gemini/antigravity` path derivation may be added.

## Discussion Criteria Mapping & Leaf Slice

<!-- binding-inventory: brain-antigravity-telemetry-wake-and-appdata -->
- `ai/scripts/diagnostics/seatCostReport.mjs`
- `test/playwright/unit/ai/scripts/diagnostics/seatCostReport.spec.mjs`
- `test/playwright/unit/ai/scripts/diagnostics/fixtures/seatCost/syntheticFixtures.mjs`
<!-- /binding-inventory -->

Delivers the complete Option C code slice graduated from `neomjs/neo` Discussion `#21469` (`#1853` AC-1..AC-5); `#1853` retains AC-6 (operational scorecard comments).

## The Fix

1. Register `'antigravity': true` in `LEDGER_READERS`, add `'gemini': null` to `WARM_WINDOWS` (renders `unmeasured`, never an invented TTL), and extend `classifyProviderFamily` to match Gemini model identifiers while preserving non-Gemini partner classification (`claude`/`gpt`).
2. Add pure/read-only Antigravity readers (`parseAntigravityTranscript`, `readAntigravitySummariesDb`, `readAntigravityConversationModel`, `collectAntigravityRecords`) using lazy `better-sqlite3` with `{readonly: true, fileMustExist: true}` and graceful fallback when optional SQLite files are absent.
3. Aggregate `(root + child)` `calls` (`rootCalls` + `childCalls`), `freshInput`, `cacheRead`, `coldPrefills` (`freshInput > 50_000` with `cacheRead === 0` or `cacheRead / (freshInput + cacheRead) < 0.1`), `compactions` (`CHECKPOINT` steps), and `output` per UTC day, resolving recursive child lineage via `parent_conversation_id` / `conversation_parents` and optional per-seat `#<workspaceFilter>` matching on shared `conversation_summaries.db` stores.
4. Accept explicit `--antigravity-root <dir>` and repeatable `--antigravity-seat <seat>=<dir>[#<workspaceSubstr>]` flags with zero `os.homedir()` `.gemini/antigravity` derivation, and skip default `os.homedir()` Kimi/OpenCode lookups when only explicit Antigravity flags are supplied.
5. Add `redactSensitiveTokens` to scrub session UUIDs and local filesystem paths / `file://` URIs from rendered Markdown, `--json`, seat labels, and CLI error output.
6. Add deterministic synthetic JSONL + temp SQLite unit tests in `seatCostReport.spec.mjs` (`syntheticFixtures.mjs`), updating the conscious-update `LEDGER_READERS` pin.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `LEDGER_READERS` & `WARM_WINDOWS` (`ai/scripts/diagnostics/seatCostReport.mjs:73-94`) | `neomjs/neo` `D#21469` & `#1853` | Sets `'antigravity': true` in `LEDGER_READERS` and `'gemini': null` in `WARM_WINDOWS`; `classifyProviderFamily` maps `gemini`/`google`/`carbon`/`neon`/`hydrogen`/`argon` to `'gemini'` (`gaps>warm` renders `unmeasured`). | Unknown/missing model string returns `null` family (`unmeasured`); non-Gemini models in Antigravity (`claude`/`gpt`) classify to their own provider family. | Yes (JSDoc + `--help`) | `seatCostReport.spec.mjs` (`LEDGER_READERS` pin, coverage line, provider classification) |
| `parseAntigravityTranscript(content, options)` (new export in `seatCostReport.mjs`) | `#1853` AC-1 & live `transcript.jsonl` schema V-B-A | Parses `PLANNER_RESPONSE` and `CHECKPOINT` JSONL steps into usage records (`freshInput`, `cacheRead`, `output`, `coldPrefills` where `freshInput > 50_000` and `cacheRead === 0` or hit ratio `< 0.1`, `compactions: 1` on `CHECKPOINT`, `role: 'root' \| 'child'`). | Malformed JSON lines, non-token step types (`USER_INPUT`, `GENERIC`, `SYSTEM_MESSAGE`), or invalid timestamps are skipped cleanly. | Yes (JSDoc) | `seatCostReport.spec.mjs` parser & cold-prefill/compaction boundary arms |
| `collectAntigravityRecords(rootDir, options)` & SQLite helpers (new exports in `seatCostReport.mjs`) | `#1853` AC-1, AC-2 | Lazily loads `better-sqlite3` and opens `conversation_summaries.db` and `conversations/<id>.db` with `{readonly: true, fileMustExist: true}`; resolves recursive `parent_conversation_id` / `conversation_parents` `(root + child)` lineage, optional `workspaceFilter`, and `gen_metadata` model strings. | Missing `conversation_summaries.db` or `conversations/<id>.db` returns zero/partial metadata without throwing or creating `-wal`/`-shm` files, falling back to `brain/*/.system_generated/logs/transcript.jsonl` scan. | Yes (JSDoc) | `seatCostReport.spec.mjs` synthetic SQLite + missing-DB fallback + zero-WAL check |
| CLI flags `--antigravity-root <dir>` & `--antigravity-seat <seat>=<dir>[#<filter>]` (`seatCostReport.mjs`) | `#1853` AC-3 & `ADR-0019` §3 B5 | Reads Antigravity ledger(s) strictly from explicit CLI paths with zero `os.homedir()` `.gemini/antigravity` derivation; renders the main per-day table plus an `## Antigravity (root + child)` breakdown (`calls (root+child)`, `fresh input`, `cache read`, `cold prefills (>50K)`, `compactions`, `output`) and includes `rootCalls`, `childCalls`, `coldPrefills`, `compactions` in `--json`. | Missing value or malformed `--antigravity-seat` spec exits 1 with redacted error + usage; when only Antigravity flags are passed, default Kimi/OpenCode `os.homedir()` paths are not probed. | Yes (`USAGE` / `--help`) | `seatCostReport.spec.mjs` CLI `--antigravity-root`, `--antigravity-seat`, `--json`, `--help` arms |
| `redactSensitiveTokens(text)` (new export in `seatCostReport.mjs`) | `#1853` AC-4 | Scrubs UUIDs (`<redacted-uuid>`), `file://` URIs, and local Unix/Windows filesystem paths (`<redacted-path>`) across seat labels, Markdown output, `--json` output, and CLI error messages. | Non-sensitive tokens (dates, counts, harness/model names, CLI flags) pass through unchanged. | Yes (JSDoc) | `seatCostReport.spec.mjs` UUID & filesystem path redaction arms |

## Acceptance Criteria

- [ ] `ai/scripts/diagnostics/seatCostReport.mjs` sets `'antigravity': true` in `LEDGER_READERS` and `'gemini': null` in `WARM_WINDOWS`, aggregating `(root + child)` `freshInput`, `cacheRead`, `coldPrefills` (`>50K`), `compactions`, and `output` in both Markdown and `--json` output.
- [ ] Lazy `better-sqlite3` opens `conversation_summaries.db` and optional `conversations/<id>.db` strictly with `{readonly: true, fileMustExist: true}`, gracefully returning partial/zero metrics when optional SQLite files are absent and creating zero `-wal`/`-shm` writes.
- [ ] Zero new `os.homedir()` `.gemini/antigravity` derivations are introduced in `seatCostReport.mjs`; explicit `--antigravity-root` / `--antigravity-seat` flags work standalone without requiring Kimi/OpenCode directories.
- [ ] Output (Markdown, `--json`, and CLI errors) redacts session UUIDs and local filesystem paths.
- [ ] Deterministic synthetic JSONL + temp SQLite unit tests pass via `npm run test-unit -- test/playwright/unit/ai/scripts/diagnostics/seatCostReport.spec.mjs`, and `node ./ai/scripts/lint/lint-script-plane.mjs` + `node ./ai/scripts/lint/check-aiconfig-antipatterns.mjs` pass.

## Avoided Traps

- Never open live `~/.gemini/antigravity/conversation_summaries.db` in tests or commit projected real-ledger telemetry (`neomjs/neo#16707` cycle-3 privacy boundary); synthesize JSONL + SQLite fixtures in `os.tmpdir()` inside `beforeAll` and clean up in `afterAll`.
- Never skip `CHECKPOINT` steps in `transcript.jsonl`: each `CHECKPOINT` step records a compaction event AND carries its summarization call's `input_tokens`, `cache_read_tokens`, and `output_tokens`.
- Never introduce a 5th `os.homedir()` path derivation (`ADR-0019` §3 B5).

## Out of Scope

- Ongoing `>=48h` operational-recovery scorecard comments and post-`v14.0` matched-task A/B runs (remain on open parent/host #1853).
- Wake-hold age diagnostics (#1854) and per-seat `appDataDir` SSOT wiring (#1855).

## Sweep Attestation

- Live latest-open + closed sweep: checked latest 20 open + open and closed search (`seatCostReport OR antigravity ledger`, `Out of Scope` / `Avoided Traps` / `Contract Ledger` read on #1853, #1854, #1855, #1846, and `neomjs/neo#16707`) at `2026-11-19T14:53Z`; `160` open (red — split directly from #1853 so #1853 stays open for AC-6's multi-day scorecard); no duplicate code leaf found.
- A2A in-flight claim sweep: scanned latest 30 `status: 'all'` messages (`2026-11-19T14:52Z`); zero `[lane-claim]` or `[lane-intent]` on `seatCostReport.mjs` or #1853.
- MC sweep: `seatCostReport.mjs antigravity brain 1853`, `Antigravity quota drain cold prefills transcript.jsonl conversation_summaries.db seatCostReport` + `ask_knowledge_base` (`neomjs/neo#16707` surfaced and read), no conflicting prior decision.
- Own-assignment sweep: `0` open issues assigned to `@neo-gemini-maryam` in `neomjs/neo-agent-brain`.
- Structure map: `npm run ai:structure-map -- --files --loc` verified `ai/scripts/diagnostics/seatCostReport.mjs` (`267` codeLoc, `38` files in `ai/scripts/diagnostics`).

## Related

- Parent / Scorecard Host: #1853
- Source Discussion: `neomjs/neo` Discussion `#21469`
- Post-`v14.0` Follow-Up: `neomjs/neo-agent-skills#423`
- Prior Art: `neomjs/neo#16707`

Origin Session ID: 095892f3-027f-47aa-8092-a13ae31f72ca
Retrieval Hint: "seatCostReport antigravity LEDGER_READERS CHECKPOINT coldPrefills conversation_summaries.db redactSensitiveTokens"

## Timeline

- 2026-11-19T14:56:04Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-11-19T14:56:06Z @neo-gemini-maryam added the `enhancement` label
- 2026-11-19T14:56:06Z @neo-gemini-maryam added the `ai` label
- 2026-11-19T14:56:06Z @neo-gemini-maryam added the `model-experience` label
- 2026-11-19T14:56:06Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-19T14:56:48Z @neo-gemini-maryam added parent issue #1853
- 2026-11-19T14:56:51Z @neo-gemini-maryam cross-referenced by #1853
- 2026-11-19T15:05:38Z @neo-gemini-maryam cross-referenced by PR #1862
- 2026-11-19T16:20:29Z @tobiu referenced in commit `084f7c3` - "feat(diagnostics): add Antigravity root+child ledger reader to seatCostReport (#1861) (#1862)"
- 2026-11-19T16:20:31Z @tobiu closed this issue

