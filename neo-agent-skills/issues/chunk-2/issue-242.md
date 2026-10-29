---
id: 242
title: 'check-substrate-size budgets 24,576 B while Antigravity truncates at 24,000 B including the RULE path wrapper'
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-29T18:17:42Z'
updatedAt: '2026-10-29T19:21:18Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/242'
author: neo-gemini-maryam
commentsCount: 1
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
closedAt: '2026-10-29T19:21:18Z'
---
# check-substrate-size budgets 24,576 B while Antigravity truncates at 24,000 B including the RULE path wrapper

Design authority: `@neo-opus-grace` (`neomjs/neo` PR `#20176` review `5702707571` `[TOOLING_GAP]`) and Maryam's empirical Antigravity `<RULE[path]>` truncation receipt (`neomjs/neo#20048`).

## Context

In `neomjs/neo#20048` (`PR #20176`), we observed empirically inside an active Antigravity session (`@neo-gemini-maryam`) that `neomjs/neo/AGENTS.md` at `24,270` bytes was silently truncated at the tail by `354` bytes (`<truncated 354 bytes>`), cutting off the last bullet of `§edge_case_triggers` (`Wake/Heartbeat → run the cycle`).

## The Problem

- `scripts/check-substrate-size.mjs:30` and `scripts/generate-agents-md.mjs:45` both set `PER_FILE_LIMIT_BYTES = 24576` (`24 KiB`), documented as *"Antigravity's documented per-file hard limit"*.
- In reality, Antigravity caps each `<RULE[/absolute/path/to/AGENTS.md]>` block at **`24,000` decimal bytes** including the `<RULE[...]>\n` wrapper tag (`84` bytes for `/Users/tobiasuhlig/.neo-ai/agents/neo-gemini-maryam/neomjs/neo/AGENTS.md`, leaving `23,916` content bytes: `24,270 - 354 = 23,916`).
- Because `24,270 <= 24,576`, `check-substrate-size.mjs` reported `✅ AGENTS.md : 24270 bytes · limit 24576 · headroom 306` while Antigravity was already truncating `354` bytes from the bottom of `AGENTS.md` on every turn.

## The Architectural Reality

- `scripts/check-substrate-size.mjs` (`PER_FILE_LIMIT_BYTES = 24576`) and `scripts/generate-agents-md.mjs` (`PER_FILE_LIMIT_BYTES = 24576`) share the byte-budget constant by value (`scripts/test-substrate-size.mjs` and `scripts/test-generate-agents-md.mjs` test it).
- Because the `<RULE[/abs/path]>` wrapper length varies slightly by seat path (~80–100 bytes), the content ceiling for an Antigravity-loaded file must stay safely below `24,000` decimal bytes minus a conservative wrapper margin (e.g. `23,800` bytes).

## The Fix

Lower `PER_FILE_LIMIT_BYTES` in `scripts/check-substrate-size.mjs` and `scripts/generate-agents-md.mjs` (and their contract tests `scripts/test-substrate-size.mjs` and `scripts/test-generate-agents-md.mjs`) from `24576` (`24 KiB`) to `23800` bytes (`24,000` decimal-byte Antigravity `<RULE[path]>` cap minus `200` B wrapper budget), and update the JSDoc to record the `24,000` decimal-byte `<RULE[path]>` empirical measurement.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `scripts/check-substrate-size.mjs` (`PER_FILE_LIMIT_BYTES`) | `neomjs/neo` PR `#20176` review `5702707571` (`[TOOLING_GAP]`) | Enforce `23800` B per-file content limit so Antigravity's `24,000` decimal-byte `<RULE[path]>` cap is never breached silently | Non-Antigravity targets share the tighter ceiling unless per-harness limits are split | JSDoc on `PER_FILE_LIMIT_BYTES` | L1: `node scripts/test-substrate-size.mjs` |
| `scripts/generate-agents-md.mjs` (`PER_FILE_LIMIT_BYTES`) | Same | Refuse to emit any variant `> 23800` B | Same | JSDoc on `PER_FILE_LIMIT_BYTES` | L1: `node scripts/test-generate-agents-md.mjs` |

Decision Record impact: aligned-with `0007-agents-md-compaction-taxonomy.md`.

## Acceptance Criteria

- [ ] AC-1 `PER_FILE_LIMIT_BYTES` in `scripts/check-substrate-size.mjs` and `scripts/generate-agents-md.mjs` is updated to `23800` (with JSDoc documenting the `24,000` decimal-byte `<RULE[path]>` wrapper calculation: `24,270 B - 354 B truncated = 23,916 B content + 84 B wrapper`).
- [ ] AC-2 `scripts/test-substrate-size.mjs` and `scripts/test-generate-agents-md.mjs` pass against the updated limit, and all emitted repository × audience variants pass `< 23800` B.
- [ ] AC-3 `package.json` and `package-lock.json` version bumped with full `npm test` green.

## Out of Scope

- Splitting separate numeric limits for Claude Code vs Antigravity (`limitConfirmed: false` on `.claude/CLAUDE.md` remains separate).

## Avoided Traps

- Assuming `24 KB` in vendor documentation means `24 * 1024 = 24,576` content bytes when the harness actually enforces `24,000` decimal bytes including its `<RULE[/abs/path]>` XML wrapper.

## Related

- `neomjs/neo#20048` (`PR #20176` review `5702707571`), `neomjs/neo-agent-skills#240` (`PR #241`).
- Live latest-open sweep: checked latest open issues in `neomjs/neo-agent-skills` at `2026-10-29T18:17:12Z`; no equivalent found.
- A2A in-flight claim sweep: checked latest messages via `list_messages({status: 'all', limit: 5})` at `2026-10-29T18:17:15Z`; no competing claim.
- Memory Core rationale sweep: ran `query_raw_memories`; no prior issue for the `24,000` decimal-byte wrapper cap.
- Structure map: N/A (existing `scripts/check-substrate-size.mjs` and `scripts/generate-agents-md.mjs`, no new `.mjs` files).

Origin Session ID: cf3d43bb-b204-4223-9012-afca3ab562a0
Retrieval Hint: "check-substrate-size PER_FILE_LIMIT_BYTES 24576 24000 Antigravity RULE wrapper truncation"

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cf3d43bb-b204-4223-9012-afca3ab562a0`. 🧭

## Timeline

- 2026-10-29T18:17:44Z @neo-gemini-maryam added the `bug` label
- 2026-10-29T18:17:44Z @neo-gemini-maryam added the `ai` label
- 2026-10-29T18:17:44Z @neo-gemini-maryam added the `model-experience` label
- 2026-10-29T18:17:45Z @neo-gemini-maryam added the `agent-os` label
- 2026-10-29T18:19:22Z @neo-gemini-maryam cross-referenced by PR #20176
- 2026-10-29T18:19:24Z @neo-gemini-maryam cross-referenced by PR #241
- 2026-10-29T18:30:45Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-29T18:38:00Z

### Ticket Intake (`self-authored-carve.md` Stage 2 + Drift Probe)

- **Drift probe (`git log origin/dev --since="2026-10-29T18:17:42Z" --name-only`):** Only PR `#241` (`agents-md/sections/*`, `context-recovery-workflow.md`, `package.json` `0.6.5`) landed since issue creation; zero intersection with `scripts/check-substrate-size.mjs` or `scripts/generate-agents-md.mjs`.
- **Stage 2 Prescription Check:**
  - `Prescription checked: scripts/check-substrate-size.mjs & scripts/generate-agents-md.mjs — owns the concern` (both scripts define `PER_FILE_LIMIT_BYTES`, shared by value so `check-substrate-size.mjs` ships its own constant when invoked against consumer repositories).
  - Verified all generated `AGENTS.md` variants against `23800` B (`neo`/`skills`/`institution`/`devindex` maintainer: `22,740` B; `neo-agent-brain` / `ALL_REPOS` maintainer: `23,387` B; live `neomjs/neo/AGENTS.md`: `23,285` B). Note: `devindex/AGENTS.md` is still on the pre-#20176 `24,272` B copy and will surface when `devindex` refreshes its `AGENTS.md` or bumps its pinned baseline tag.

Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cf3d43bb-b204-4223-9012-afca3ab562a0`. 🧭

- 2026-10-29T18:42:08Z @neo-gemini-maryam cross-referenced by PR #243
- 2026-10-29T18:58:51Z @neo-gemini-maryam cross-referenced by PR #106
- 2026-10-29T19:21:18Z @tobiu referenced in commit `0c8831b` - "fix(agentos): lower PER_FILE_LIMIT_BYTES to 23,800 B for Antigravity RULE cap (#242) (#243)"
- 2026-10-29T19:21:18Z @tobiu closed this issue

