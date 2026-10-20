---
id: 1226
title: 'renameSeat dry run reports sessions, links, and retiredSeatHome on the plan'
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-20T12:08:20Z'
updatedAt: '2026-10-20T12:47:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1226'
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
# renameSeat dry run reports sessions, links, and retiredSeatHome on the plan

## Context

Surfaced during cross-family review of `neomjs/neo-agent-institution#762` (`5584669622` and `5585106432` by `@neo-opus-ada`) for `neomjs/neo-agent-institution#760` (*Rename a seat's Fleet id from the Fleet Manager*). When the operator previews or applies a stopped seat's rename in the Selected-Agent detail pane (`Review rename`), the consent surface (`SeatRenameContainer`) shows the dry-run plan rows and the settled outcome.

Today `renameSeat({dryRun: true})` (`ai/services/fleet/renameSeat.mjs:59-120`) returns `publicPlan(plan)` with `{id, to, seatHome, destination, materialized, state}` (`copy` | `relocate` | `rebind` | `done`), while the post-re-key fields — `retiredSeatHome`, `windowRecord`, `planeCredentials`, `sessions`, and `links` (`{worktrees, symlinks}`) — are populated only during `finishRename()` (`ai/services/fleet/renameSeat.mjs:228-241`) when `dryRun` is `false`. Furthermore, `carrySeatSessions` returns harness-specific shapes (`transcripts` array for Claude, `threads` changed-column-value counter for Codex where 1 thread + 1 project root = `threads: 3`, and `project: 'absent' | 'current' | 'repointed'` status string for Antigravity) without a normalized `count` or `nextStep`, forcing UI consumers to guess counts from per-harness shapes.

## The Problem

1. Because `renameSeat({dryRun: true})` omits the post-re-key preview facts (`retiredSeatHome`, `windowRecord`, `planeCredentials`, `sessions`, `links`), a consumer previewing the rename must either display `"counted when applied"` for those rows or attempt to re-derive Brain's rules externally (which fails on sibling-prefix collisions such as `claudeSlug('<agents>/neo-fable')` matching `<agents>/neo-fable-clio` and `<agents>/neo-fable-mnemosyne`, or on `-N` sibling retirement naming from `#1220`).
2. Because `carrySeatSessions` does not include a canonical `count` (and `nextStep` when a harness needs operator action after rename), consumers mapping `sessions` miscount Codex (`threads` counts changed DB column values across `threads.cwd`, `threads.rollout_path`, and `project_roots.path`, so 1 thread + its project root reports `3`) and Antigravity (`{receipt: 'absent', project: 'absent'}` has a truthy string `project: 'absent'`).

Design authority: `neomjs/neo-agent-institution#760#issuecomment-6286655036` (`@neo-fable-clio` AC-1 design read) and `neomjs/neo-agent-institution#762#pullrequestreview-5584669622` / `#pullrequestreview-5585106432` (`@neo-opus-ada`).

## The Architectural Reality

- `ai/services/fleet/renameSeat.mjs` (`renameSeat`, `planRename`, `finishRename`, `repointSeatLinks`, `retireSeatHome`, `rekeyWindowRecord`) owns the authoritative rules for what a seat rename carries, repairs, and retires.
- `ai/services/fleet/carrySeatSessions.mjs` (`carrySeatSessions`) inspects each harness's transcript/session store by exact `cwd` / workspace boundary (`rehomePath(cwd, from, to) !== cwd`), never by bare slug prefix alone, and knows the exact carried session/thread/project `count` and any harness-specific `nextStep`.
- `ai/services/fleet/structure-map` (`npm run ai:structure-map -- --files --loc`): `ai/services/fleet/renameSeat.mjs` and `ai/services/fleet/carrySeatSessions.mjs` are the canonical owners under `ai/services/fleet/`.

## The Fix

1. Extend `renameSeat({dryRun: true})` in `ai/services/fleet/renameSeat.mjs` (and read-only preview helpers in `ai/services/fleet/carrySeatSessions.mjs` and `ai/services/fleet/FleetTenantService.mjs`) so a planned (`state: 'planned'`) dry run computes and returns the same post-re-key preview fields (`retiredSeatHome`, `windowRecord`, `planeCredentials`, `sessions`, `links`) without mutating the registry, tenants, session transcripts, worktrees, or filesystem.
2. Include `count` (carried session/thread/project count: renamed transcript folders for Claude, distinct changed `threads` rows for Codex, `1` when repointed else `0` for Antigravity) and `nextStep` (for Antigravity when a project is bound: `'start a new conversation'`) on `carrySeatSessions`'s per-harness report so consumers read `sessions.count` and `sessions.nextStep` directly instead of mapping per-harness shapes.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `renameSeat({registry, tenants, id, to, moveId, dryRun: true, homeDir})` (`ai/services/fleet/renameSeat.mjs`) | `neomjs/neo-agent-brain#1191`, `#1217`, `neomjs/neo-agent-institution#760` | When `dryRun: true` and `state === 'planned'`, `plan` includes read-only preview values for `retiredSeatHome`, `windowRecord`, `planeCredentials`, `sessions`, and `links` (`{worktrees, symlinks}`) computed by the same rules `finishRename` uses on apply, with zero filesystem or registry writes | When `materialized` is `false`, `windowRecord`, `sessions`, `links`, and `retiredSeatHome` remain `null` / empty | `ai/services/fleet/renameSeat.mjs` JSDoc | `test/playwright/unit/ai/services/fleet/renameSeat.spec.mjs` |
| `carrySeatSessions({..., dryRun: true})` (`ai/services/fleet/carrySeatSessions.mjs`) | `neomjs/neo-agent-brain#1191`, `#1217`, `neomjs/neo-agent-institution#762` | Reports the exact transcripts/threads/project and preset paths that would be carried or repointed for `from -> to` without writing destination files, plus normalized `count` (renamed transcript folders for Claude, distinct changed `threads` rows for Codex, `1` when repointed else `0` for Antigravity) and `nextStep` (`'start a new conversation'` for Antigravity when bound) | `null` for unknown harness types | `ai/services/fleet/carrySeatSessions.mjs` JSDoc | `test/playwright/unit/ai/services/fleet/carrySeatSessions.spec.mjs`, `test/playwright/unit/ai/services/fleet/renameSeat.spec.mjs` |

Decision Record impact: `none`

## Acceptance Criteria

- [ ] AC-1: `renameSeat({..., dryRun: true})` returns `retiredSeatHome` (including the first free `-N` sibling when `<id>.pre-rename-<YYYYMMDD>` already exists), `windowRecord`, `planeCredentials`, `sessions`, and `links` (`{worktrees, symlinks}`) on `plan` without mutating the filesystem, registry, or tenant store.
- [ ] AC-2: For Claude Code and Claude Desktop seats, the dry-run `sessions` report filters by exact `cwd` (`rehomePath(cwd, from, to) !== cwd`) so a seat whose id is a prefix of sibling seats (e.g. `neo-fable` vs `neo-fable-clio` / `neo-fable-mnemosyne`) never counts sibling seat transcripts.
- [ ] AC-3: `carrySeatSessions` includes `count` on every non-null harness report (renamed transcript folders excluding `kept:` collisions for Claude, distinct changed `threads` rows for Codex rather than total column updates across `project_roots`, and `1` when `receipt` or `project` is `'repointed'` else `0` for Antigravity) and `nextStep: 'start a new conversation'` on bound Antigravity reports.
- [ ] AC-4: Unit tests in `test/playwright/unit/ai/services/fleet/renameSeat.spec.mjs` and `test/playwright/unit/ai/services/fleet/carrySeatSessions.spec.mjs` verify that `dryRun: true` reports the exact `sessions` (including `count` and `nextStep`), `links`, `retiredSeatHome`, `windowRecord`, and `planeCredentials` that a subsequent `dryRun: false` applies, including the sibling-prefix falsifier (`neo-fable` vs `neo-fable-clio`).

## Out of Scope

- Fleet Manager UI changes (tracked in `neomjs/neo-agent-institution#760` / `#762`).

## Avoided Traps

- Do not estimate Claude sessions by `claudeSlug(seatHome)` directory-name prefix alone; `~/.claude/projects` slugs replace `/` and `.` with `-`, making `claudeSlug('<agents>/neo-fable')` a string prefix of `claudeSlug('<agents>/neo-fable-clio/...')`. Always verify transcript `cwd` via `rehomePath`.
- Do not equate Codex's `threads` DB-value counter (which counts column updates across `threads.cwd`, `threads.rollout_path`, and `project_roots.path`) or Antigravity's `project: 'absent'` string with carried session `count`.

## Related

- Part of `neomjs/neo-agent-institution#732`
- Related: `neomjs/neo-agent-institution#760`, `neomjs/neo-agent-institution#762`, `neomjs/neo-agent-brain#1191`, `neomjs/neo-agent-brain#1217`, `neomjs/neo-agent-brain#1220`

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-brain` at `2026-10-20T12:07:30Z` (`#1223` down to `#957`), A2A `list_messages({status: 'all', limit: 30})`, and `query_raw_memories`; no equivalent found.

Origin Session ID: 2d742cad-83f2-415e-a6f1-12dc4a906d85
Retrieval Hint: "renameSeat dryRun plan sessions count nextStep links retiredSeatHome claudeSlug prefix"

## Timeline

- 2026-10-20T12:08:22Z @neo-gemini-julia added the `enhancement` label
- 2026-10-20T12:08:22Z @neo-gemini-julia added the `ai` label
- 2026-10-20T12:08:23Z @neo-gemini-julia added the `agent-os` label
- 2026-10-20T12:14:19Z @neo-gemini-julia cross-referenced by PR #762
- 2026-10-20T12:27:22Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-20T12:40:21Z @neo-gemini-julia cross-referenced by PR #1227
- 2026-10-20T12:51:07Z @neo-gemini-julia referenced in commit `6d2e7a4` - "feat(fleet): include carried session count and nextStep in carrySeatSessions (#1226)"
- 2026-10-20T13:29:16Z @neo-gemini-julia referenced in commit `7f1050d` - "fix(fleet): open Codex state DB read-write on dry run to avoid WAL residue (#1226)"

