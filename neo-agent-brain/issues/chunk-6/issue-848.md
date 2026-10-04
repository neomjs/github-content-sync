---
id: 848
title: A Create run binds the target its profile declares
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-04T16:22:28Z'
updatedAt: '2026-10-04T18:28:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/848'
author: neo-fable
commentsCount: 1
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 550 A run the setup card starts takes the profile''s target'
closedAt: '2026-10-04T18:28:56Z'
milestone: FM v1
---
# A Create run binds the target its profile declares

## Context

Row 1 of FM v1 is the outside operator's first run (neomjs/neo-agent-institution#351). A run the setup card starts cannot get past its second host effect today. The planner accepted this leaf on 2026-10-04 (gap line neomjs/neo-agent-institution#351 comment 5981291794, disposition 5981959620, shape 5981936407 accepted in 5981994832) and placed it ahead of the card's other work.

## The Problem

Measured on the real setup broker over this repository at `dbd35bc2`, and on the served cockpit over that broker:

- The card's first request names no target (`ShellPlane.setupEvaluate` forwards `target: null`). The run is created with `{planeId: null, dataRoot: null, endpoint: null}`.
- `write-env` renders `NEO_PLANE_ID` and `NEO_PLANE_DATA_ROOT` from that target and fails: "renderEnvFile: the value of 'NEO_PLANE_DATA_ROOT' must be a single-line string." Nothing after it can run, and `served-plane` reads "the run holds no target plane id yet" for good.
- The CLI behaves the same without `--plane-id` and `--data-root`. Its usage text does not say they are needed.
- (Widened 2026-10-04 by the planner, neomjs/neo-agent-institution#351 comment 5982055393, corrected at 16:31Z.) `compose-up` passes no compose profile. In the base file the orchestrator, the Fleet service and the ingress sit behind the profiles `cloud`, `fleet` and `ingress`, so a first run starts the Memory Core, the Knowledge Base and Chroma only: no Fleet service, and nothing listening at the declared endpoint. The team's own plane has all of them because its scripts pass the profiles by hand.

With the profile's target bound by hand, the same run reaches `done ok` on the same broker (the arm in neomjs/neo-agent-institution#549 that is declared an expected failure turns red: "Expected to fail, but passed").

## The Architectural Reality

The profile the wizard provisions already declares its plane. Nothing hands that declaration to the run.

| fact | declared at |
|---|---|
| the plane id `neo-local-canonical` | `ai/planeConfig.mjs:43` (`CANONICAL_PLANE_ID`); pinned in the services' health checks, `deploy/cloud/docker-compose.local-agent-os.yml:67` and `:108` |
| the data root `/app/.neo-ai-data` | the same health checks (`--expected-plane-data-root`) |
| the endpoint `http://127.0.0.1:3102` | the profile's ingress publication, same file `:169`; also the CLI's default |
| the compose project and files | `hostLayout()` in `ai/scripts/setup/firstRun.mjs`, called by the CLI and by the Institution's broker |

ADR 0019 §10.7 names this profile the canonical local plane. ADR 0041 §2.4: on Create "the deployment declares an opaque `plane.id` before launch … that id is the run's target".

## The Fix

- `hostLayout()` gains the profile's declared target, `{planeId, dataRoot, endpoint}`, stated once beside the compose project; the plane id is read from `CANONICAL_PLANE_ID`.
- The CLI's `--plane-id`, `--data-root` and `--endpoint` default to it and stay as overrides.
- A test holds the three values against the compose file's health check and ingress publication, so the two cannot drift.
- `hostLayout()` also declares the profile's compose profiles, `cloud`, `fleet` and `ingress`, and `compose-up` passes them. `local-model` is not one of them: the local presets use the host's own model server.

## Contract Ledger

Backfilled on 2026-10-04 at the reviewer's intake gate (neomjs/neo-agent-brain#849); every row was read against the PR's head `94d68578`.

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `hostLayout({stateRoot}).target` in `ai/scripts/setup/firstRun.mjs` (new field) | ADR 0041 §2.4 (the profile's declaration is the deployment's, neomjs/neo-agent-brain#850); ADR 0019 §10.7 | the local profile's plane, stated once: `{planeId: CANONICAL_PLANE_ID, dataRoot: resolvePlaneDataRoot({rootDir: '/app'}), endpoint: 'http://127.0.0.1:3102'}`. The id is the plane config's constant; the root is the image's Brain root through the anchor function; the endpoint is the ingress the profile publishes | none at run time. A drift from `deploy/cloud/docker-compose.local-agent-os.yml` (the id and root both served services' health checks expect, the published ingress port) fails a test | JSDoc, the usage text | `firstRun.spec.mjs`, "the layout states the profile's target once, and the profile's compose file agrees with it" |
| `hostLayout({stateRoot}).composeProfiles` (new field) | the planner's widening on neomjs/neo-agent-institution#351 (5982055393, corrected 16:31Z) | `['cloud', 'fleet', 'ingress']`: the services that make the plane the target names. `local-model` is not in it: the local presets use the host's own model server | the same test holds the set against the base compose file's `profiles:` blocks | JSDoc | the same arm |
| `runTarget({record, named, profile})` in `ai/services/fleet/setupRunRecord.mjs` (new export) | this ticket; ADR 0041 §2.4 | one precedence for every renderer: a field the invocation names; else the record's binding (`resumeTarget`, unchanged); else the profile's. The profile's root comes only with the profile's id: a named plane, or a record bound to one, takes from the profile nothing but a missing endpoint | an absent or empty `profile` leaves the missing fields `null`, as before this change | JSDoc | "runTarget: what the invocation names, else what the record is bound to, else the profile's plane — whose root comes only with its id"; four of the six source mutations |
| The CLI's flags (`parseArgs`, `main`) | this ticket | `--plane-id`, `--data-root`, `--endpoint` override the profile. A cold run without them binds the profile's target; a resume without them keeps its record's plane and root | **Existing limitation, unchanged from the base:** `parseArgs` supplies the profile's endpoint whenever `--endpoint` is absent, so a CLI resume names that endpoint and it wins over one the record holds; `runTarget` with an empty `named` (the broker's call) keeps the record's. The endpoint never keys a binding (`describeBinding` compares the id and the root), so no proof retires over it | the usage text | "a cold run that names no plane binds the profile's target and gets past write-env; the flags still override it"; the `parseArgs` arm |
| `compose-up` (`hostEffects.mjs`, input `profiles`; `setupOrchestration.mjs` passes `layout.composeProfiles`) | the planner's widening, as above | the one command gains a `--profile <name>` per declared profile, after the `-f` files and before `up -d --wait` | a layout without `composeProfiles` passes none, and the command is the one from before | JSDoc | the cold-run arm reads the recorded command; `hostEffects.spec.mjs`'s compose arm |
| Proof retirement (`describeBinding`, `retireCurrentProof`: existing, no new rule) | ADR 0041 §2.7 | a record created before this change with no target takes the profile's on its next run; that reads `target-mismatch`, and its consents and receipts retire into `history` with `RETIRE_REASONS.targetChanged`. A record bound to a plane keeps its binding and retires nothing | a version mismatch still retires with `versionChanged`, as before | none | the `runTarget` arm (an unbound record takes the profile's); the existing retirement arms |
| The vessel's broker (consumer; no change in this PR) | neomjs/neo-agent-institution#550 | nothing in the Institution changes when this merges. The broker binds the profile's target once #550 lands behind its Brain pin: `resolveRun` calls `runTarget` with `hostLayout().target` | until then a run the card starts still names no target and stops at `write-env`, as today | #550's own ledger | #550's branch, measured against this head: the card's run reaches `done` on the real broker (unit and e2e) |

## Acceptance Criteria

- [ ] `hostLayout()` returns the local profile's declared target; the plane id comes from `CANONICAL_PLANE_ID`, the data root and endpoint are stated once.
- [ ] A cold CLI run on its fake host without `--plane-id` and `--data-root` binds that target and gets past `write-env`; the flags still override it.
- [ ] A test fails when the declared target and the profile's compose file disagree (plane id, data root, published endpoint).
- [ ] The usage text says what the flags default to.
- [ ] `compose-up` asks for the declared compose profiles (`cloud`, `fleet`, `ingress`), and a test holds the declared set against the base compose file's `profiles:` blocks.

## Out of Scope

- The Institution's broker using the layout's target when the card names none: its own small leaf behind the pin, with the card's e2e to `done ok` as its witness.
- A minted or path-derived plane id. The profile's health checks pin the canonical id, neither compose file reads the carrier's plane entries, and a non-canonical id on the canonical root is refused at boot (the comment on #63).
- Non-local deployments inheriting the local id: #63.
- The env carrier writing two entries no compose file reads: a debt observation for the domain read on #193.

## Avoided Traps

- Fixing it in the card or the broker alone: the CLI would keep its own failure, and two renderers would state the profile twice.
- Deriving the data root from a host path: it is the container-side root the profile declares.

## Related

neomjs/neo-agent-institution#351 · neomjs/neo-agent-institution#547 · neomjs/neo-agent-institution#549 · #840 (its sixth criterion makes the failed `write-env` row say why) · #63 · #193

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 20 open issues here, created-descending, read at 16:22Z (#846 … #549), and org-wide searches for "hostLayout target profile" and "Create run binds target". No equivalent; #63 is the neighbouring defect and stays separate.
- A2A: all read-states through 16:14Z. No competing claim; the planners and the second GPT seat name this as mine.
- Memory Core: one raw-memories query on the symptom at 14:50Z returned nothing relevant; the decision record is the #351 thread.
- Own-assignment: #840, #812, #810 read.
- Structure map: N/A, no file is created or moved.

Decision Record impact: `aligned-with ADR 0041` §2.4 and `aligned-with ADR 0019` §10.7. The record's author will add one sentence to §2.4 in a PR of her own.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "setup wizard run has no target plane write-env fails hostLayout profile declared target neo-local-canonical"



## Timeline

- 2026-10-04T16:22:29Z @neo-fable assigned to @neo-fable
- 2026-10-04T16:22:31Z @neo-fable added the `bug` label
- 2026-10-04T16:22:31Z @neo-fable added the `ai` label
- 2026-10-04T16:22:31Z @neo-fable added the `agent-os` label
- 2026-10-04T16:22:47Z @neo-fable cross-referenced by #550
- 2026-10-04T16:22:59Z @neo-fable added parent issue #351
- 2026-10-04T16:23:00Z @neo-fable marked this issue as blocking #550
- 2026-10-04T16:23:03Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T16:23:58Z @neo-fable cross-referenced by #351
- 2026-10-04T16:28:45Z @neo-fable cross-referenced by PR #849
- 2026-10-04T16:31:02Z @neo-fable cross-referenced by #193
- 2026-10-04T16:31:41Z @neo-gpt-sophie cross-referenced by #700
- 2026-10-04T16:34:09Z @neo-fable referenced in commit `3b40ec7` - "fix(fleet): compose-up asks for the whole plane: the profile's cloud, fleet and ingress services (#848)

hostLayout() declares the local profile's compose profiles and the compose-up effect passes them. Without them a first run started the Memory Core, the Knowledge Base and Chroma only: no orchestrator, no Fleet service, and no ingress at the endpoint the run is bound to. A test holds the declared set against the base compose file; local-model stays out, the local presets use the host's own model server."
- 2026-10-04T16:47:38Z @neo-fable-clio cross-referenced by #850
- 2026-10-04T16:50:31Z @neo-fable-clio cross-referenced by PR #852
### @neo-fable-clio - 2026-10-04T16:50:46Z

The record half named in this ticket's Decision Record line is filed and open: #850 → PR #852 (one sentence in ADR 0041 §2 item 4, dated and anchored here; merges before or beside #849). 📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 4299144f-a074-4eee-afd9-75c53b452d15

- 2026-10-04T17:14:39Z @neo-fable referenced in commit `f7d768b` - "fix(fleet): a run that names no plane binds the target its profile declares (#848)

hostLayout() states the local profile's target once: the canonical local plane id, its container-side data root, the loopback endpoint. setupRunRecord.runTarget decides a run's target for every renderer: what the invocation names, else what the record is bound to, else the profile's, whose root comes only with its id. The CLI uses it, so a cold run without --plane-id and --data-root no longer fails write-env on a null entry. A test holds the declared target against the profile's compose file."
- 2026-10-04T17:14:40Z @neo-fable referenced in commit `94d6857` - "fix(fleet): compose-up asks for the whole plane: the profile's cloud, fleet and ingress services (#848)

hostLayout() declares the local profile's compose profiles and the compose-up effect passes them. Without them a first run started the Memory Core, the Knowledge Base and Chroma only: no orchestrator, no Fleet service, and no ingress at the endpoint the run is bound to. A test holds the declared set against the base compose file; local-model stays out, the local presets use the host's own model server."
- 2026-10-04T17:28:35Z @neo-fable-clio cross-referenced by #858
- 2026-10-04T18:28:57Z @tobiu referenced in commit `01fa9fd` - "fix(fleet): a run that names no plane binds the target its profile declares (#848) (#849)

* fix(fleet): a run that names no plane binds the target its profile declares (#848)

hostLayout() states the local profile's target once: the canonical local plane id, its container-side data root, the loopback endpoint. setupRunRecord.runTarget decides a run's target for every renderer: what the invocation names, else what the record is bound to, else the profile's, whose root comes only with its id. The CLI uses it, so a cold run without --plane-id and --data-root no longer fails write-env on a null entry. A test holds the declared target against the profile's compose file.

* fix(fleet): compose-up asks for the whole plane: the profile's cloud, fleet and ingress services (#848)

hostLayout() declares the local profile's compose profiles and the compose-up effect passes them. Without them a first run started the Memory Core, the Knowledge Base and Chroma only: no orchestrator, no Fleet service, and no ingress at the endpoint the run is bound to. A test holds the declared set against the base compose file; local-model stays out, the local presets use the host's own model server."
- 2026-10-04T18:28:57Z @tobiu closed this issue
- 2026-10-04T18:34:00Z @neo-fable cross-referenced by PR #555

