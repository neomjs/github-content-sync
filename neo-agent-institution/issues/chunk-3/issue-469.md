---
id: 469
title: 'The Institution pins Brain 804356b: the next-action holder and awaitingMerge'
state: CLOSED
labels:
  - enhancement
  - ai
  - dependencies
assignees:
  - neo-opus-ada
createdAt: '2026-10-02T20:32:00Z'
updatedAt: '2026-10-02T20:54:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/469'
author: neo-opus-ada
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
closedAt: '2026-10-02T20:54:08Z'
---
# The Institution pins Brain 804356b: the next-action holder and awaitingMerge

## Context

The Institution pins Brain `8a078f0` (#460 via PR #462). Brain `dev` is at `804356b`, four merges later.

| Brain merge | What reaches the Institution |
|---|---|
| neomjs/neo-agent-brain#780 | The open-work projection names each open PR's next-action holder (`holder` on every summary) and adds `awaitingMerge`. This is #449's source. |
| neomjs/neo-agent-brain#781 | Message ticket references resolve to typed graph nodes (Memory Core). |
| neomjs/neo-agent-brain#778 | The wake delivery reader reads a declared records directory (`fleet.wakeReceiverRecordsDir`). |
| neomjs/neo-agent-brain#777 | The quality floor grounds a canonical Neo path by its identity (diagnostics). |

## Delta census

`git diff --stat 8a078f0 804356b -- src package.json`, read by rows: **no rows.** No wire method, contract module or package entry moves. The projection's new fields reach the cockpit at runtime, through the existing `fleetOpenWork` verb.

## The Fix

- **Point the pin** at `804356b` in `package.json`, `package-lock.json` and `.github/workflows/ci.yml` (the Brain checkout `ref`).
- **Reinstall** the dependency (`rm -rf node_modules/neo-agent-brain && npm install`) and check a marker from the delta: the installed `ai/services/fleet/openWorkHolder.mjs` exports `holderOf`.
- **Run CI's jobs:** unit, components and E2E, plus the cross-repository job (Brain at the pin, one shared physical engine, `NEO_AGENTOS_RUNTIME_ROOT`).

## Acceptance Criteria

- [ ] AC-1: All three places name `804356b`, and the installed dependency carries `openWorkHolder.mjs`.
- [ ] AC-2: Both CI jobs are green: the unit, components and E2E job, and the Brain-bound unit contract with the E2E list.

## Out of Scope

- Consuming `holder` and `awaitingMerge` (#449).
- #547's post-merge check (neomjs/neo-agent-brain#503).

## Sweeps

- **Live latest-open:** the latest 20 open Institution issues at 2026-10-02T20:31Z, plus a "pin Brain" search and the open PRs that touch `package.json`. No equivalent.
- **A2A in-flight:** inbox 20:00Z–20:31Z. No pin claim.
- **Own-assignment:** #449 consumes this pin.

Origin Session ID: 6f7d14a3-e126-4b47-888f-fc28c748ae83


## Timeline

- 2026-10-02T20:32:01Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-02T20:32:02Z @neo-opus-ada added the `enhancement` label
- 2026-10-02T20:32:02Z @neo-opus-ada added the `ai` label
- 2026-10-02T20:32:03Z @neo-opus-ada added the `dependencies` label
- 2026-10-02T20:37:15Z @neo-opus-ada cross-referenced by PR #470
- 2026-10-02T20:54:08Z @tobiu referenced in commit `999fb37` - "feat(deps): pin Brain 804356b — the next-action holder and awaitingMerge (#469) (#470)

Brain 8a078f0 -> 804356b in package.json, the lock and ci.yml's Brain checkout ref. Carries neomjs/neo-agent-brain#780 (holderOf and awaitingMerge on the open-work projection, #449's source), #781, #778 and #777. No src or package.json row moves in the Brain delta."
- 2026-10-02T20:54:08Z @tobiu closed this issue

