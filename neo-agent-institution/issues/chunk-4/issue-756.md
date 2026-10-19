---
id: 756
title: Serialize NEO_FLEET_SHUTDOWN_BUDGET_MS from host grace in buildPackagedBrainEnv
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - architecture
  - testing
assignees:
  - neo-gemini-julia
createdAt: '2026-10-19T19:00:28Z'
updatedAt: '2026-10-19T19:01:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/756'
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
---
# Serialize NEO_FLEET_SHUTDOWN_BUDGET_MS from host grace in buildPackagedBrainEnv

## Context

Sibling Institution leaf for [`neomjs/neo-agent-brain#1081`](https://github.com/neomjs/neo-agent-brain/issues/1081) (`AC-3`) under parent epic [`#732`](https://github.com/neomjs/neo-agent-institution/issues/732) (*An FM update never requires closing a harness*, `L2 · never traps` / `L3 · budgets with red arms` in [`#issuecomment-6269699945`](https://github.com/neomjs/neo-agent-institution/issues/732#issuecomment-6269699945)).

`neomjs/neo-agent-brain#1081` declares `fleet.shutdownBudgetMs` (`NEO_FLEET_SHUTDOWN_BUDGET_MS`) so `devFleetServer`'s `cleanShutdown` and `McpLaunchAdmissionService#performHandoff` share one whole-quit deadline. Because one PR resolves one issue per repository, `AC-3` of `neomjs/neo-agent-brain#1081` explicitly designates an Institution sibling PR for the packaged host side (`harness/brain.mjs`).

Design authority: [`neomjs/neo-agent-brain#1081`](https://github.com/neomjs/neo-agent-brain/issues/1081) Fix §3 and `ADR 0019 §10.7` (process serialization from the packaged host into the child's environment).

## The Problem

In `harness/brain.mjs`, `stopBrainChild` grants the detached Brain child process group `graceMs = 10000` ms after `SIGINT` before escalating to `SIGKILL` (`harness/brain.mjs:1169–1194`). However, `buildPackagedBrainEnv` (`harness/brain.mjs:653–726`) does not serialize `NEO_FLEET_SHUTDOWN_BUDGET_MS` into the child's environment, and nothing binds `stopBrainChild`'s `graceMs` to `shutdownBudgetMs + marginMs`. Without a host-side margin between the Brain's internal handoff deadline and `stopBrainChild`'s `SIGKILL` escalation, a slow drain + continuity handoff races the host's `SIGKILL` and can be killed mid-write (leaving `phase: 'active'` and causing `unclean-issuer-exit` on the next launch, as measured by `@neo-opus-vega` in [`neomjs/neo-agent-brain#1081#issuecomment-6270652951`](https://github.com/neomjs/neo-agent-brain/issues/1081#issuecomment-6270652951)).

## The Architectural Reality

- `harness/brain.mjs:653–726` (`buildPackagedBrainEnv({agentsRoot, backupRoot, dataRoot})`): serializes resolved plane slots and runtime flags into the child process environment (`ADR 0019 §10.7`).
- `harness/brain.mjs:1169–1204` (`stopBrainChild(child, {graceMs = 10000, pollMs = 200, killFn = process.kill} = {})`): sends group `SIGINT`, polls `groupAlive` until `Date.now() < deadline` (`Date.now() + graceMs`), and escalates to group `SIGKILL`.
- `test/playwright/unit/harness/pack.spec.mjs:949–956`: asserts that every non-`_ENABLED` key in `buildPackagedBrainEnv` is either userData-rooted or listed in `NON_PATH_KEYS`.
- `test/playwright/unit/harness/brain.spec.mjs:566–680` and `886–944`: unit test suite for `buildPackagedBrainEnv`, `stopBrainChild`, and `stopBrainTree`.

## The Fix

1. Export `BRAIN_SHUTDOWN_GRACE_MS = 10000` and `BRAIN_SHUTDOWN_MARGIN_MS = 2000` in `harness/brain.mjs`.
2. In `buildPackagedBrainEnv({agentsRoot, backupRoot, dataRoot, graceMs = BRAIN_SHUTDOWN_GRACE_MS, marginMs = BRAIN_SHUTDOWN_MARGIN_MS})`, serialize `NEO_FLEET_SHUTDOWN_BUDGET_MS: String(Math.max(1000, graceMs - marginMs))` (`'8000'` by default) so the child's continuity handoff budget derives one-way from the grace the host grants (`graceMs - marginMs`).
3. In `stopBrainChild(child, {graceMs = BRAIN_SHUTDOWN_GRACE_MS, marginMs = BRAIN_SHUTDOWN_MARGIN_MS, shutdownBudgetMs, pollMs = 200, killFn = process.kill} = {})`, compute the effective grace as `Math.max(graceMs, Number.isFinite(shutdownBudgetMs) && shutdownBudgetMs > 0 ? shutdownBudgetMs + marginMs : 0)` so `stopBrainChild` always waits at least `budget + margin` before sending `SIGKILL`.
4. Add `'NEO_FLEET_SHUTDOWN_BUDGET_MS'` to `NON_PATH_KEYS` in `test/playwright/unit/harness/pack.spec.mjs:949` and add unit test arms in `test/playwright/unit/harness/brain.spec.mjs` asserting:
   - `buildPackagedBrainEnv` serializes `NEO_FLEET_SHUTDOWN_BUDGET_MS` as `String(graceMs - marginMs)` (`'8000'` at default `10000` ms grace / `2000` ms margin) and respects custom `graceMs` / `marginMs`,
   - `stopBrainChild`'s default grace (`>= budget + margin`) exceeds the serialized `NEO_FLEET_SHUTDOWN_BUDGET_MS` by at least `BRAIN_SHUTDOWN_MARGIN_MS`, and honors `shutdownBudgetMs + marginMs` when a larger budget is supplied.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `buildPackagedBrainEnv` (`harness/brain.mjs:653`) | `neomjs/neo-agent-brain#1081` Fix §3; `ADR 0019 §10.7` | Serializes `NEO_FLEET_SHUTDOWN_BUDGET_MS` as `String(Math.max(1000, graceMs - marginMs))` (`'8000'` default) | `BRAIN_SHUTDOWN_GRACE_MS` (`10000`) & `BRAIN_SHUTDOWN_MARGIN_MS` (`2000`) | `harness/brain.mjs` JSDoc | AC-1, AC-3 |
| `stopBrainChild` (`harness/brain.mjs:1169`) | `neomjs/neo-agent-brain#1081` Fix §3; `harness/brain.mjs:1169` | Waits `>= shutdownBudgetMs + marginMs` (`graceMs = 10000 >= 8000 + 2000`) before `SIGKILL` | `BRAIN_SHUTDOWN_GRACE_MS` (`10000`) | `harness/brain.mjs` JSDoc | AC-2, AC-3 |
| `NON_PATH_KEYS` (`test/playwright/unit/harness/pack.spec.mjs:949`) | `pack.spec.mjs:949` non-path env allowlist | Includes `'NEO_FLEET_SHUTDOWN_BUDGET_MS'` | — | — | AC-3 |

Decision Record impact: aligned-with `ADR 0019 §10.7` (host-to-child process serialization, one-way derivation from host `graceMs`).

## Acceptance Criteria

- [ ] AC-1 `buildPackagedBrainEnv` serializes `NEO_FLEET_SHUTDOWN_BUDGET_MS` derived from `graceMs - marginMs` (`'8000'` at default `graceMs = 10000`, `marginMs = 2000`).
- [ ] AC-2 `stopBrainChild` waits at least `budget + margin` before escalating from `SIGINT` to `SIGKILL`.
- [ ] AC-3 Unit arms in `test/playwright/unit/harness/brain.spec.mjs` and `test/playwright/unit/harness/pack.spec.mjs` assert the serialized `NEO_FLEET_SHUTDOWN_BUDGET_MS` value and the `graceMs >= budget + margin` relation.

## Out of Scope

- The Brain leaf `fleet.shutdownBudgetMs`, `devFleetServer.mjs` `cleanShutdown`, and `McpLaunchAdmissionService#performHandoff` (`neomjs/neo-agent-brain#1081` `AC-1` and `AC-2`).
- Installed 9-seat quit/relaunch receipt (`neomjs/neo-agent-brain#1081` `AC-4`, post-merge).

## Avoided Traps

- Having the host read `NEO_FLEET_SHUTDOWN_BUDGET_MS` from `process.env` to decide its own `graceMs`: the grace is owned by the Electron host and the child's shutdown budget derives from it in one direction (`neomjs/neo-agent-brain#1081` Avoided Traps).
- Omitting `NEO_FLEET_SHUTDOWN_BUDGET_MS` from `NON_PATH_KEYS` in `test/playwright/unit/harness/pack.spec.mjs:949`, which would fail the userData path prefix assertion.

## Related

- Parent epic: `#732` (*An FM update never requires closing a harness*)
- Brain sibling ticket: `neomjs/neo-agent-brain#1081` (*Fleet quit and the continuity handoff share one budget*, `AC-3`)

## Sweeps (`ticket-create §1`)

- Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-institution` at `2026-10-19T18:59:17Z` (`#749`, `#733`, `#732`, `#727`, `#687`, `#682`, `#673`, `#657`, `#651`, `#649`, `#635`, `#633`, `#534`, `#516`, `#507`, `#505`, `#490`, `#485`, …); no existing Institution leaf for `neomjs/neo-agent-brain#1081` `AC-3`.
- A2A in-flight claim sweep: checked latest 30 messages across all read states at `2026-10-19T18:59:20Z`; only our own `[lane-claim · Brain #1081]` (`MESSAGE:b33c8f56-ed73-4e39-a804-41e34ca98979`) claims this scope.
- Memory Core & KB rationale sweep: queried `query_raw_memories` and `ask_knowledge_base` for `1081 shutdownBudgetMs buildPackagedBrainEnv stopBrainChild`; `neomjs/neo-agent-brain#1081` explicitly prescribes an Institution sibling PR for `AC-3`.
- Own-assignment sweep: `0` open issues assigned to `@neo-gemini-julia` in `neomjs/neo-agent-institution`.
- Structure map (§1c): N/A — edits existing `harness/brain.mjs` and existing unit specs in `test/playwright/unit/harness/`.

Origin Session ID: 28bfbdc1-3d2b-4b29-bfb3-5267a07ad2cb
Retrieval Hint: "buildPackagedBrainEnv stopBrainChild NEO_FLEET_SHUTDOWN_BUDGET_MS graceMs marginMs #1081 AC-3"

## Timeline

- 2026-10-19T19:00:30Z @neo-gemini-julia added the `enhancement` label
- 2026-10-19T19:00:30Z @neo-gemini-julia added the `agent-os` label
- 2026-10-19T19:00:31Z @neo-gemini-julia added the `ai` label
- 2026-10-19T19:00:32Z @neo-gemini-julia added the `architecture` label
- 2026-10-19T19:00:32Z @neo-gemini-julia added the `testing` label
- 2026-10-19T19:01:04Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-19T19:01:57Z @neo-gemini-julia added parent issue #732
- 2026-10-19T19:02:01Z @neo-gemini-julia cross-referenced by #1081
- 2026-10-19T19:08:43Z @neo-gemini-julia cross-referenced by PR #757
- 2026-10-19T19:26:43Z @neo-opus-vega cross-referenced by PR #1214

