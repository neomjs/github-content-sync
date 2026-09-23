---
id: 435
title: An unconfigured plane's CI-ingest and defect-digest lanes fail every tick
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-09-23T13:48:01Z'
updatedAt: '2026-09-23T17:11:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/435'
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
closedAt: '2026-09-23T17:11:17Z'
---
# An unconfigured plane's CI-ingest and defect-digest lanes fail every tick

## Context

Two independent sightings. The first is finding F3 on the #253 cut receipt: on the local plane, `ci-failure-ingest` exits 1 without a GitHub credential. The second is Clio's fresh-plane first-run receipt (2026-09-23 12:32Z, compose project `fm-fresh-small`, the b99ea11 images). Within its first 6 s, `defect-ledger-digest` exited 1 on `no plane is configured (AiConfig.fleet.planeBase)` and `ci-failure-ingest` exited 1 on `set GH_TOKEN or GITHUB_TOKEN`. The second sighting promotes F3 from a defect-note (`ticket-create` §1e).

## The Problem

Each due tick spawns the child: hourly for the ingest, every 6 h for the digest. The child throws, and the supervisor records a `failed` outcome and logs an ERROR. A plane that simply is not configured for these lanes therefore reads as a plane whose lanes are broken, and an operator on first run cannot tell "missing credential" from "crash". Neither lane escalates (`ESCALATING_TASK_OUTCOMES` holds only `backup`), so the cost is false health and log noise, not paging.

## The Architectural Reality

- Both lanes are supervised one-shots with a JSON outcome contract. They are defined in `ai/daemons/orchestrator/taskDefinitions.mjs` (`'defect-ledger-digest'`, `'ci-failure-ingest'`), both with `captureStdoutJson: true`. `ai/daemons/orchestrator/scheduling/registry.mjs` schedules them with no `enables` gate.
- The supervisor already has the right outcome. `ProcessSupervisorService#classifySuccessfulChildOutcome` maps a child that exits 0 and prints `{deferred: true, reason}` to `skipped`, with that reason code, and does not refresh `lastSuccessAt` (the generic deferred envelope).
- `ai/scripts/maintenance/ingestCiFailures.mjs` `main()` resolves the credential first. `resolveGithubToken()` in `ai/services/ingestion/githubActions.mjs` throws when none is set. Only then does it open the plane mailbox, and `openPlaneMailbox()` throws on an empty `AiConfig.fleet.planeBase`.
- `ai/scripts/diagnostics/defectObservations.mjs` throws on `--digest` when no plane base is configured.
- Structure map: `npm run ai:structure-map -- --files --loc` exit 0 at `36c4aac`. No file is added or moved; the owners stay `ai/scripts/maintenance`, `ai/scripts/diagnostics` and `ai/daemons/orchestrator`.

## The Fix

In both scripts, an absent prerequisite ends the tick as deferred: print `{"deferred": true, "reason": "<code>"}` and exit 0. The codes are `github-token-unset` and `plane-base-unset`. The scheduler, the registry and the supervisor stay unchanged.

Inputs that are present but wrong keep failing: a rejected token, an unreachable plane, `plane init failed`. Deferral means "not configured", never "broken". There is no new AiConfig leaf and no new env read, because `resolveGithubToken()` and `AiConfig.fleet.planeBase` are already the resolution points and the scripts only test what they return.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `ci-failure-ingest` stdout + exit code | the deferred envelope in `classifySuccessfulChildOutcome` | absent credential or plane base → `{deferred, reason}`, exit 0 | present-but-failing input → exit 1 as today | script JSDoc | script arms for each code, plus the failing control |
| `defect-ledger-digest` stdout + exit code | same | absent plane base → `{deferred, reason: 'plane-base-unset'}`, exit 0 | `--local` and `--plane-base` unchanged | script JSDoc | script arm |

## Decision Record impact

`none`. It uses an existing envelope and adds no config.

## Acceptance Criteria

- [ ] With no GitHub credential, `ingestCiFailures.mjs --json` prints `{"deferred": true, "reason": "github-token-unset"}`, exits 0, and makes no GitHub or mailbox call.
- [ ] With a credential but no plane base (and neither `--plane-base` nor `--local`), it prints `{"deferred": true, "reason": "plane-base-unset"}` and exits 0.
- [ ] `defectObservations.mjs --digest` with no plane base prints `{"deferred": true, "reason": "plane-base-unset"}` and exits 0.
- [ ] A present-but-failing input still exits 1 (control arm, e.g. `plane init failed`).
- [ ] On an unconfigured plane the supervisor records both lanes as `skipped` with those reason codes, shown through `classifySuccessfulChildOutcome` on the lanes' actual stdout.

## Out of Scope

- Setting the credential or the plane base in the first-run recipe (neomjs/neo#18965).
- Passing a GitHub credential to the orchestrator in the `deploy/cloud` compose files, which is why the lane has none on the local plane. That is a plane-credential decision under neomjs/neo#18965; after this ticket such a plane reads `skipped: github-token-unset` instead of failing.
- kb-server and mc-server ignoring SIGTERM, the other open defect-note from the same cut.

## Avoided Traps

- **An `enables` gate in the registry.** It would hide the lane on an unconfigured plane, so the snapshot shows nothing to configure. A `skipped` outcome with a reason code names the missing input where operators already look. The gate would also add an AiConfig-derived predicate (the ADR-0019 surface) for a question the scripts already answer.
- **Exit 0 with no output.** Indistinguishable from a real success, and it would refresh `lastSuccessAt`.

## Related

#253 (F3) · #425 (its Out of Scope names this note) · #327 (the CI failure ingestor) · neomjs/neo#18965

Live latest-open sweep: checked the latest 20 open issues at 2026-09-23T13:45:57Z and re-checked the latest 5 at 13:47:20Z (newest #434); no equivalent.
A2A in-flight sweep: 30 messages across read states (newest 13:40Z). The only one on this scope is Clio's 12:33Z receipt, with no claim.
MC sweep: "fresh plane lane exits 1 every tick ingestCiFailures GH_TOKEN defect digest planeBase not configured deferred skipped", 6 results, all unrelated; no prior decision.
Own-assignment sweep: 19 open assigned to me; none covers these lanes (#427 is the PR-event source, #49 the heap guard).

Origin Session ID: 3be453e4-8b04-4865-be62-4cff34f4e0c6

Retrieval Hint: `query_raw_memories("unconfigured plane lane deferred envelope ingestCiFailures defectObservations digest plane base GitHub token")`


## Timeline

- 2026-09-23T13:48:01Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-23T13:48:03Z @neo-opus-ada added the `bug` label
- 2026-09-23T13:48:04Z @neo-opus-ada added the `ai` label
- 2026-09-23T13:48:05Z @neo-opus-ada added the `agent-os` label
- 2026-09-23T14:14:06Z @neo-opus-ada cross-referenced by PR #436
- 2026-09-23T17:11:17Z @tobiu referenced in commit `bb76e7e` - "Merge pull request #436 from neomjs/ada/435-unconfigured-lanes-defer

fix(orchestrator): an unconfigured plane's CI-ingest and defect-digest lanes defer (#435)"
- 2026-09-23T17:11:17Z @tobiu closed this issue

