---
id: 763
title: SeatRenameContainer reads sessions.count and sessions.nextStep from renameSeat plan
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T13:28:49Z'
updatedAt: '2026-10-20T21:49:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/763'
author: neo-gemini-julia
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-20T21:49:01Z'
---
# SeatRenameContainer reads sessions.count and sessions.nextStep from renameSeat plan

## Context

In #760 / #762, `apps/agentos/view/fleet/detail/SeatRenameContainer.mjs` shipped the Selected-Agent inline seat-rename flow (`Review rename` -> dry-run plan preview -> `Rename seat` confirm) before Brain's `renameSeat({dryRun: true})` (`neomjs/neo-agent-brain#1226` / `neomjs/neo-agent-brain#1227`) populated post-re-key fields on the dry-run plan or exposed a uniform `sessions.count` and `sessions.nextStep` contract across harnesses.

Once `neomjs/neo-agent-brain#1227` merges, `renameSeat` and `carrySeatSessions` return:
- `sessions.count` (`Number`): uniform carried-session count across all four supported harnesses (`claude-code` transcript files changed, `claude-desktop` session/preset/worktree files changed, `codex-desktop` distinct `threads` rows changed, `antigravity` project binding repointed `0 | 1`).
- `sessions.nextStep` (`String`, optional): operator next step when a harness leaves per-conversation state at the old path (e.g. Antigravity's `'start a new conversation'`), alongside `sessions.conversations`.
- Dry-run `plan.retiredSeatHome`, `plan.planeCredentials`, `plan.windowRecord`, `plan.sessions`, and `plan.links` computed directly from `plan.seatHome -> plan.destination` without mutating disk.

Design authority: #760, `neomjs/neo-agent-brain#1226` (`AC-3`), and Row 2 (#477 — every surface names its state with a reason and a next step).

## The Problem

`apps/agentos/view/fleet/detail/SeatRenameContainer.mjs` still carries the pre-`#1226` fallback heuristics:
1. `countCarriedSessions(plan)` (`:332–340`) does not check `Number.isFinite(sessions.count)` first; instead it falls through to `sessions.transcripts.length`, `sessions.threads` (which on Codex Desktop counts changed column values across `threads` and `project_roots` rather than distinct thread rows), or `'project' in sessions` (`sessions.project` is `'repointed' | 'current' | 'absent'`, a non-empty string that evaluates truthy even when `'absent'` or `'current'`).
2. `renderPlan(plan)` (`:383–385`) reads `plan.sessions?.note` / `plan.sessions?.presets`, neither of which `carrySeatSessions` returns (`carrySeatSessions` returns `sessions.nextStep`, `sessions.conversations`, and `sessions.records`).
3. `applyRenameResult(result)` (`:319–322`) formats the settled status line (`Renamed <from> → <to> · carried <N> sessions · retired <retiredSeatHome>`) without appending `plan.sessions?.nextStep` when the harness requires a follow-up step (such as Antigravity's `'start a new conversation'`).

## The Architectural Reality

- Consumer: `apps/agentos/view/fleet/detail/SeatRenameContainer.mjs` (`countCarriedSessions`, `renderPlan`, `applyRenameResult`).
- Producer: `neomjs/neo-agent-brain` `ai/services/fleet/carrySeatSessions.mjs` and `ai/services/fleet/renameSeat.mjs` (`neomjs/neo-agent-brain#1226` / `neomjs/neo-agent-brain#1227`), pinned in `package.json`.
- Unit spec: `test/playwright/unit/apps/agentos/view/fleet/detail/container.spec.mjs`.
- Structure map (`npm run ai:structure-map -- --files --loc`): Brain `ai/services/fleet/carrySeatSessions.mjs`; Institution `apps/agentos/view/fleet/detail/SeatRenameContainer.mjs` (no new `.mjs` files).

## The Fix

1. Bump `neo-agent-brain` in `package.json` once `neomjs/neo-agent-brain#1227` merges to `dev`.
2. In `apps/agentos/view/fleet/detail/SeatRenameContainer.mjs`:
   - `countCarriedSessions(plan)`: return `sessions.count` first when `Number.isFinite(sessions.count)` holds (preserving legacy fallback arms only when `sessions.count` is absent).
   - `renderPlan(plan)`: derive the `Harness state` row from `plan.sessions?.nextStep` / `plan.sessions?.conversations` (or `plan.sessions?.records` for Claude Desktop records repointed) before falling back to `plan.sessions?.note`.
   - `applyRenameResult(result)`: include `plan.sessions?.nextStep` in the settled status text when present (`Renamed <from> → <to> · carried <N> sessions · retired <path> · next: <nextStep>`).
3. Extend `test/playwright/unit/apps/agentos/view/fleet/detail/container.spec.mjs` to assert `sessions.count` and `sessions.nextStep` rendering across both `applyPlanResult` (dry-run preview, including Codex `threads: 3, count: 1` and Antigravity `project: 'current', count: 0, nextStep: 'start a new conversation'`) and `applyRenameResult` (settled status).

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `SeatRenameContainer#countCarriedSessions` | `SeatRenameContainer.mjs:332–340`; `neomjs/neo-agent-brain#1226` | Returns `plan.sessions.count` when `Number.isFinite(plan.sessions.count)` is true | Legacy `carriedSessions` / `transcripts.length` / `threads` fallback when `count` is absent | JSDoc on `SeatRenameContainer.mjs` | Unit (`container.spec.mjs`) |
| `SeatRenameContainer#renderPlan` (`Harness state` row) | `SeatRenameContainer.mjs:383–385`; `carrySeatSessions.mjs` | Renders `plan.sessions.conversations` / `plan.sessions.nextStep` (or Claude Desktop `records` count) | `'none'` when unmaterialized or no harness note applies | JSDoc on `SeatRenameContainer.mjs` | Unit (`container.spec.mjs`) |
| `SeatRenameContainer#applyRenameResult` | `SeatRenameContainer.mjs:294–324` | Appends ` · next: ${plan.sessions.nextStep}` when `plan.sessions?.nextStep` is a non-empty string | Existing settled text when `nextStep` is absent | JSDoc on `SeatRenameContainer.mjs` | Unit (`container.spec.mjs`) |

Decision Record impact: aligned-with ADR 0034.

## Acceptance Criteria

- [ ] AC-1: `SeatRenameContainer#countCarriedSessions` prefers `plan.sessions.count` when finite, so a Codex plan with `{threads: 3, count: 1}` reports `1` session and an Antigravity plan with `{project: 'current', count: 0}` reports `0` sessions.
- [ ] AC-2: `SeatRenameContainer#renderPlan` surfaces `plan.sessions.conversations` / `plan.sessions.nextStep` (and `plan.sessions.records` for Claude Desktop) in the `Harness state` row, and `SeatRenameContainer#applyRenameResult` includes `plan.sessions.nextStep` in the settled status text when present.
- [ ] AC-3: `test/playwright/unit/apps/agentos/view/fleet/detail/container.spec.mjs` covers both `sessions.count` and `sessions.nextStep` in the plan preview and settled outcome.

## Out of Scope

- Changing Brain's `carrySeatSessions` or `renameSeat` return shape (`neomjs/neo-agent-brain#1226` / `neomjs/neo-agent-brain#1227`).
- Modifying the shell broker (`harness/seatRenameBroker.mjs`), which already forwards `plan` unchanged.

## Avoided Traps

- Inferring Antigravity session count from `'project' in sessions` (`sessions.project` is `'absent'` or `'current'` even when `0` sessions changed).
- Counting Codex changed column values (`sessions.threads`) instead of distinct thread rows (`sessions.count`).

## Related

- Parent: #732
- Predecessor: #760 / #762
- Brain producer: `neomjs/neo-agent-brain#1226` / `neomjs/neo-agent-brain#1227` (blocked by until merged)

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` at 2026-10-20T13:27Z; no equivalent found.
A2A claim sweep: checked latest 30 messages (`status: 'all'`) at 2026-10-20T13:27Z; no competing claim on `SeatRenameContainer`.
MC sweep: `SeatRenameContainer countCarriedSessions sessions.count nextStep`, 6 results, surfaced #760/#762 and `neomjs/neo-agent-brain#1227` RA-2; no prior ticket found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-institution`, none overlapping.

Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85
Retrieval Hint: "SeatRenameContainer countCarriedSessions sessions.count nextStep #1226 #760"


## Timeline

- 2026-10-20T13:28:49Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T13:28:51Z @neo-gemini-julia added the `enhancement` label
- 2026-10-20T13:28:52Z @neo-gemini-julia added the `agent-os` label
- 2026-10-20T13:28:52Z @neo-gemini-julia added the `ai` label
- 2026-10-20T13:28:52Z @neo-gemini-julia added the `testing` label
- 2026-10-20T13:29:07Z @neo-gemini-julia added parent issue #732
- 2026-10-20T13:39:36Z @neo-opus-ada cross-referenced by PR #1227
- 2026-10-20T15:57:14Z @neo-gemini-julia cross-referenced by PR #767
- 2026-10-20T16:05:26Z @neo-gemini-julia referenced in commit `73bc778` - "fix(fleet): stamp visual baseline for SeatRenameContainer (#763)"
- 2026-10-20T19:21:06Z @neo-gemini-julia referenced in commit `5616f67` - "fix(fleet): read sessions.count and sessions.nextStep in SeatRenameContainer (#763)"
- 2026-10-20T21:33:32Z @neo-gemini-julia referenced in commit `ec5a0eb` - "fix(fleet): render Antigravity nextStep once in SeatRenameContainer plan (#763)"
- 2026-10-20T21:38:51Z @neo-gemini-julia referenced in commit `9205d13` - "chore(test): refresh visual baseline input stamp for SeatRenameContainer (#763)"
- 2026-10-20T21:49:02Z @tobiu closed this issue
- 2026-10-20T21:49:02Z @tobiu referenced in commit `33771e5` - "fix(fleet): read sessions.count and sessions.nextStep in SeatRenameContainer (#763) (#767)

* fix(fleet): read sessions.count and sessions.nextStep in SeatRenameContainer (#763)

* fix(fleet): render Antigravity nextStep once in SeatRenameContainer plan (#763)

* chore(test): refresh visual baseline input stamp for SeatRenameContainer (#763)"

