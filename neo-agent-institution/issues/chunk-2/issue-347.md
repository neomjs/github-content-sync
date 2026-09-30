---
id: 347
title: The packaged Brain places every plane member under the user data root
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-30T12:13:54Z'
updatedAt: '2026-09-30T15:29:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/347'
author: neo-opus-vega
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
closedAt: '2026-09-30T14:31:43Z'
---
# The packaged Brain places every plane member under the user data root

## Context

#345 (PR #346, merged 2026-09-30) moved the Fleet store out of the app bundle after Emmy found Sophie's registry and encrypted key inside `Neo Harness.app`, where a whole-app replacement deletes them. Grace's defect-note that day named seven more Tier-1 plane members with the same defect. The operator asked the same day that frequent FM updates never lose data.

A census of every declared plane member, run against the Brain the Institution installs (`node_modules/neo-agent-brain`) and `buildPackagedBrainEnv` at `dev@4fcc0fd`, finds **19** members still on their build-time anchor default, `<organism>/.neo-ai-data`, which sits inside the bundle:

| Config base | Unplaced members (env) |
|---|---|
| Tier-1 `ai/configBase.mjs` (8 of 13) | `NEO_AUTH_SEAT_TOKEN_REGISTRY_PATH` · `NEO_HEARTBEAT_ALIVE_PATH` · `NEO_HEARTBEAT_LOCK_PATH` · `NEO_FLEET_INSTANCE_ROOT` · `NEO_HEAP_OBSERVATION_DIR` · `NEO_DEPLOYMENT_STATE_BRIDGE_SNAPSHOT_PATH` · `NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH` · `NEO_RECOVERY_ACTUATOR_RUN_STATE_DIR` |
| memory-core (8 of 11) | `NEO_MEMORY_DB_PATH` · `NEO_RLAIF_PATH` · `NEO_AI_DAEMON_DIR` · `NEO_HOOK_PROJECTION_ROOT` · `NEO_HANDOFF_FILE_PATH` · `NEO_GOLDEN_PATH_ROUTE_ATTRIBUTION_LEDGER_DIR` · `NEO_MEMORY_LOG_PATH` · `NEO_LAZY_EDGES_QUEUE_PATH` |
| knowledge-base (2 of 2) | `NEO_KB_EMBEDDING_RESUME_STATE_DIR` · `NEO_KB_LOG_PATH` |
| neural-link (1 of 1) | `NEO_NL_LOG_PATH` |

## The Problem

A replacement of the app deletes all 19 members' data. One of them also splits the graph. `NEO_AI_DB_PATH` (Tier-1 `orchestrator.dbPath`) and `NEO_MEMORY_DB_PATH` (memory-core `storagePaths.graphProd`) declare the same file, `sqlite/memory-core-graph.sqlite`. The packaged env relocates only the first. In own mode, the orchestrator therefore maintains a graph under the user data root, while the memory-core server reads and writes a second graph inside the bundle.

The Brain already refuses this state. When `plane.dataRoot` is relocated, `assertPlaneMemberCoherence` fails boot on any member left on its anchor default (ADR 0019 §10.5, "a partially-moved plane"). The packaged profile never relocates `plane.dataRoot`, though, so that check never runs, and the partial move ships silently.

## The Architectural Reality

- `harness/brain.mjs#buildPackagedBrainEnv` is the packaged product profile. It places 11 members explicitly and does not set `NEO_PLANE_DATA_ROOT`.
- Each Brain config base exports `PLANE_MEMBER_PATHS`, and each member leaf carries its own env binding. `collectPlaneMembers` + `assertPlaneMemberCoherence` (`ai/planeConfig.mjs`) are the boot check that `BaseServer.runHealthcheckAndLogStatus` and the orchestrator daemon run.
- ADR 0019 §10.7 elects placement per profile and records no packaged-product row. Adding a durable profile is its stated revalidation trigger.

## The Fix

- `buildPackagedBrainEnv` sets `NEO_PLANE_DATA_ROOT` to the data root and places each of the 19 members beneath it. Each member keeps its canonical sub-path, except the recovery-actuator and route-attribution paths, which nest in the profile's existing `orchestrator` dir rather than `orchestrator-daemon`.
- Once `plane.dataRoot` is relocated, the Brain's own boot check guards the profile: a member added later without a placement fails boot rather than landing in the bundle.
- A unit arm runs that same check (`collectPlaneMembers` + `assertPlaneMemberCoherence`) against the packaged env for all four config bases, so CI catches an omission before any boot does.

## Acceptance Criteria

AC-1, AC-2 and AC-4 landed via #348 (merge db9d9f7, approved at 108adc0 in review 5366895891). AC-3 is the one criterion still open. Its witness is a packaged candidate built from a dev that carries db9d9f7, run in own mode under a throwaway userData and allocated ports; the installed profile, which runs Sophie against the shared plane, is never touched. That is the same bundle layout without installation, and the receipt will say so.

- [x] AC-1 `buildPackagedBrainEnv({dataRoot})` sets `NEO_PLANE_DATA_ROOT` to `dataRoot` and binds every member declared in the four config bases' `PLANE_MEMBER_PATHS` to a path beneath `dataRoot`. Both graph leaves name the same file.
- [x] AC-2 A unit arm resolves the Brain's configs under the packaged env and passes the Brain's own `assertPlaneMemberCoherence` for all four config bases. With one binding removed, it fails and names that member.
- [ ] AC-3 (post-merge, installed) An installed build in own mode writes no file under `<bundle>/Contents/Resources/organism/.neo-ai-data`.
- [x] AC-4 Both harness profiles set `NEO_AI_DEPLOYMENT_MODE` to `local`. The Brain's `cloud` default turns the localOnly lanes off (Chroma, the embed and message daemons), so an own-mode organism ran without Chroma (the packaged smoke's `chromaListening` red, found by Grace). The resolved `orchestrator.deploymentMode` reads `local` in the Brain child.

## Out of Scope

- Migrating state already written inside an installed bundle. The next replacement deletes it regardless; the first boot after this fix starts those members fresh. The Fleet store, the one with credentials, already moved via #346.
- The Brain-side ADR 0019 §10.7 row for the packaged profile, a Brain docs follow-up.
- The checkout smoke profile (`buildBrainProfile`), which #214 is changing.

## Avoided Traps

- **Setting only `NEO_PLANE_DATA_ROOT`.** Member defaults are anchor-static, so this fails boot on every member (§10.5).
- **Listing the seven Tier-1 paths from the note.** The per-server bases carry 12 more, the graph among them. The member list comes from `PLANE_MEMBER_PATHS`, never from a copy.
- **Rebasing every member generically at runtime.** §10.5 makes relocation explicit per-member placement, never an implicit cascade.

## Related

#345 · #346 · #214 · #12 · #7 · Brain `ai/planeConfig.mjs` · ADR 0019 §10.4–§10.7

Live latest-open sweep: the latest 20 open issues at 2026-09-30T12:12Z; no equivalent (#345 covers the Fleet store only). A2A claim sweep (last 30, all read states): no claim; Grace offered the broader guard to Emmy after #214. Memory Core rationale sweep: #345's session is the only prior record. Own-assignment sweep: #341, #244; neither is this.

Decision Record impact: aligned-with ADR 0019 §10.5 (per-member placement, enforced coherence).

Origin Session ID: 3589c87d-68f1-474c-94c7-b9cbc38b1f5d
Retrieval Hint: "packaged plane members user data root NEO_PLANE_DATA_ROOT assertPlaneMemberCoherence"





## Timeline

- 2026-09-30T12:13:55Z @neo-opus-vega added the `bug` label
- 2026-09-30T12:13:55Z @neo-opus-vega added the `agent-os` label
- 2026-09-30T12:13:55Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-30T12:13:55Z @neo-opus-vega added the `ai` label
- 2026-09-30T12:22:32Z @neo-opus-vega cross-referenced by PR #348
- 2026-09-30T12:24:03Z @neo-gpt-emmy cross-referenced by #345
- 2026-09-30T12:24:48Z @neo-fable-clio cross-referenced by #335
- 2026-09-30T12:32:57Z @neo-fable-clio cross-referenced by #349
- 2026-09-30T12:49:49Z @neo-opus-vega referenced in commit `108adc0` - "fix(harness): both profiles run the Brain local, so own mode supervises Chroma again (#347)

- Brain deploymentMode defaults to cloud, which turns the localOnly lanes off (Chroma, the embed and
  message daemons), and neither buildPackagedBrainEnv nor buildBrainProfile set it: a Finder-launched
  own-mode organism ran without Chroma, the smoke's chromaListening red. Both now set
  NEO_AI_DEPLOYMENT_MODE=local. Found by Grace while classifying a #214 smoke red.
- pack.spec names the key beside the authority role; brain.spec reads the resolved
  orchestrator.deploymentMode from a Brain child."
- 2026-09-30T12:58:42Z @neo-opus-grace cross-referenced by PR #350
- 2026-09-30T13:19:29Z @neo-fable-clio cross-referenced by #351
- 2026-09-30T13:34:43Z @tobiu referenced in commit `db9d9f7` - "Merge pull request #348 from neomjs/vega/347-plane-member-paths

fix(harness): the packaged Brain places every plane member under the user data root (#347)"
- 2026-09-30T14:31:43Z @tobiu closed this issue
- 2026-09-30T14:32:41Z @neo-opus-vega cross-referenced by #358
- 2026-09-30T14:33:20Z @neo-opus-grace cross-referenced by #359
- 2026-09-30T15:05:55Z @neo-fable-clio cross-referenced by #361
- 2026-09-30T15:08:28Z @neo-fable-clio cross-referenced by PR #363
- 2026-09-30T15:47:23Z @neo-opus-vega cross-referenced by #641
- 2026-09-30T15:52:12Z @neo-gpt cross-referenced by PR #364

