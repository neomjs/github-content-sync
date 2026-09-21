---
id: 115
title: Harness and test loaders follow the Fleet wire to Brain's canonical home
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-09-05T12:32:09Z'
updatedAt: '2026-09-05T13:12:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/115'
author: neo-fable-clio
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
closedAt: '2026-09-05T13:12:44Z'
---
# Harness and test loaders follow the Fleet wire to Brain's canonical home

## Context

Brain PR #326 (merged 2026-09-05T12:10:47Z as `a9e010d`, neomjs/neo-agent-brain#217) retired `ai/services/fleet/fleetWireMethods.mjs`: the wire vocabulary now lives in `src/fleet/contract/wire.mjs`, the launcher's credential classification in the private `ai/services/fleet/fleetLaunchContract.mjs`, the cockpit identifiers in `src/fleet/contract/cockpit.mjs`. The Institution binds to Brain by filesystem path from `NEO_AGENTOS_RUNTIME_ROOT`, and its `Explicit Brain contract` job checks out Brain `ref: dev` (`.github/workflows/ci.yml:73-78`). Since that merge the job reds on every PR — first witness PR #114 at `fa5d435`: `test/playwright/unit/harness/fleetCapability.spec.mjs:18` → `loadAgentOsModule('ai/services/fleet/fleetWireMethods.mjs')` → module not found (operator-pasted CI log, 12:2xZ). The Electron launcher carries the same import at boot (`harness/brain.mjs:108`), so a dev harness over a Brain `dev` checkout boots without a Fleet wire. The producer landed before the consumer adoption (#43, unassigned, no PR); the coordinated-landing gate in #326's body was not exercised.

## The Problem

Six loader sites name the retired path, or `fleetCockpitStatus.mjs` for identifiers that moved with it. #43 is the real adoption — the immutable Brain package pin, the App-Worker import from `neo-agent-brain/fleet-contract`, the four twins and the parity spec deleted — a lane with an install-footprint decision in it. Until it lands, CI is red for everyone and the local launcher is dead. This leaf re-points the path-bound consumers at the canonical homes and changes nothing else, so the seam #43 replaces stays exactly where it is.

## The Architectural Reality

- `harness/brain.mjs:104-112` `loadFleetRuntimeContracts` imports three modules from the runtime root: `ai/graph/normalizeAgentIdentityNodeId.mjs`, `ai/services/fleet/fleetLaunchContract.mjs` (already, for `probeExistingFleetServer` / `resolveFleetBearer`) and the retired wire module; it maps `FLEET_CREDENTIAL_METHODS` from the wire module, which now lives in `fleetLaunchContract.mjs:9` (frozen from `FLEET_WIRE_METHODS`; #217's ledger row "Launcher credential-method classification").
- `harness/fleetCapability.mjs:142-160` throws a `TypeError` without a string array of credential methods — the boot cannot fail soft.
- Retired-path loaders: `test/playwright/unit/harness/fleetCapability.spec.mjs:18`, `test/playwright/unit/harness/brain.spec.mjs:35`, `test/playwright/unit/apps/agentos/fleet/fleetTransport.integration.spec.mjs:34`, `test/playwright/e2e/agentos/authenticatedFleetHarness.mjs:15` (collected by the cross-repo job's `--list`), `test/playwright/unit/apps/agentos/view/fleet/util/kindRegistry.spec.mjs:29` (`FLEET_COCKPIT_EVENT_TYPES`, now in `src/fleet/contract/cockpit.mjs`).
- `fleetVocabularyParity.spec.mjs` self-gates on `apps/agentos/config` under the runtime root (absent post-split): skipped, not failing; #43 deletes it.
- `loadAgentOsModule` (Engine `test/playwright/fixtures.mjs:27-37`) is a plain `import()` under the runtime root — there is no aliasing layer to patch.

## The Fix

One path table, no behavior change:

| site | was | is |
|---|---|---|
| `harness/brain.mjs:108` | `ai/services/fleet/fleetWireMethods.mjs` | `src/fleet/contract/wire.mjs`; `FLEET_CREDENTIAL_METHODS` read from the already-imported `fleetLaunchContract` |
| `fleetCapability.spec.mjs:18` | the wire module incl. the credential methods | wire from `src/fleet/contract/wire.mjs`, credential methods from `ai/services/fleet/fleetLaunchContract.mjs` |
| `brain.spec.mjs:35` · `fleetTransport.integration.spec.mjs:34` · `authenticatedFleetHarness.mjs:15` | `ai/services/fleet/fleetWireMethods.mjs` | `src/fleet/contract/wire.mjs` |
| `kindRegistry.spec.mjs:29` | `ai/services/fleet/fleetCockpitStatus.mjs` | `src/fleet/contract/cockpit.mjs` |

The JSDoc of `loadFleetRuntimeContracts` names the two homes. The parity gate and the four twins are untouched.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `loadFleetRuntimeContracts` (`harness/brain.mjs`) | #217's canonical homes: `src/fleet/contract/wire.mjs`, private `ai/services/fleet/fleetLaunchContract.mjs` | the same contract object with the same keys, read from the moved homes | unchanged: a root without the modules rejects the boot with the named reason | JSDoc | `brain.spec.mjs` + `fleetCapability.spec.mjs` through the real modules at Brain `dev ≥ a9e010d` |
| the five test loaders | the same homes | the same symbols from the moved homes | unchanged | — | the `Explicit Brain contract` job green at the PR head against Brain `dev` |

Decision Record impact: aligned-with ADR 0038 (the pure-client boundary the homes implement); none amended.

## Acceptance Criteria

- [ ] `harness/brain.mjs` loads the wire vocabulary from `src/fleet/contract/wire.mjs` and `FLEET_CREDENTIAL_METHODS` from `ai/services/fleet/fleetLaunchContract.mjs`; `createFleetCapability` accepts the result against Brain `dev@a9e010d` (`fleetCapability.spec.mjs` and `brain.spec.mjs` through the real modules — red on `dev`, green on the branch).
- [ ] Every `loadAgentOsModule` of a retired path is re-pointed (the six sites above); `git grep -n 'ai/services/fleet/fleetWireMethods\|fleetCockpitStatus.mjs' -- harness test` returns nothing outside the self-gated `fleetVocabularyParity.spec.mjs`, which stays as it is (#43 deletes it). *(AC restated 2026-09-05 12:36Z: the first wording forgot the gated spec.)*
- [ ] The `Explicit Brain contract` job is green at the PR head against Brain `ref: dev`.
- [ ] No twin, no parity-spec, no package change — #43's scope stays whole.

## Out of Scope

#43 (the Brain package pin, the App-Worker import from `neo-agent-brain/fleet-contract`, twin and parity-spec deletion) · #17 (dissolving `loadFleetRuntimeContracts`) · pinning the workflow's Brain ref to a pre-#326 SHA — rejected: it would green CI while the local launcher stays broken, and it freezes the cross-repo witness on a stale producer.

## Related

#43 (the adoption this leaf precedes) · #17 (the demotion this seam eventually leaves) · neomjs/neo-agent-brain#217 / PR #326 (`a9e010d`) · PR #114 (the first red witness) · neomjs/neo-agent-brain#206.

Sweeps: live latest-open (20 issues read 2026-09-05T12:24Z) — #43 is the adoption, no leaf for the path repair; A2A last-12 all-states (12:25Z) — no claim on #43 or this surface (Euclid on neomjs/neo#5822, Grace on #17920); Memory Core `query_raw_memories` — the index reported `allWritesSemanticallyQueryable: false` this morning, recorded as an index miss; own-assignment: #113, #112, #103, #21 open; #17 (unassigned, mine) is the same seam's demotion — cited, not merged.

Origin Session ID: 3a507e14-4d80-4512-bf0c-68ac34638902
Retrieval Hint: "Institution harness loadFleetRuntimeContracts fleetWireMethods retired src/fleet/contract/wire.mjs Explicit Brain contract red"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 3a507e14-4d80-4512-bf0c-68ac34638902

## Timeline

- 2026-09-05T12:32:09Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-05T12:32:10Z @neo-fable-clio added the `bug` label
- 2026-09-05T12:32:10Z @neo-fable-clio added the `agent-os` label
- 2026-09-05T12:32:10Z @neo-fable-clio added the `ai` label
- 2026-09-05T12:32:10Z @neo-fable-clio added the `testing` label
- 2026-09-05T12:35:29Z @neo-fable-clio cross-referenced by PR #116
- 2026-09-05T13:12:44Z @tobiu referenced in commit `1fb92af` - "Merge pull request #116 from neomjs/agent/115-harness-follows-brain-wire

fix(harness): the launcher and the test loaders follow the Fleet wire to Brain's canonical home (#115)"
- 2026-09-05T13:12:44Z @tobiu closed this issue

