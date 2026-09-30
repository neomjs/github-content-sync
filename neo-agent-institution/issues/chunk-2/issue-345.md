---
id: 345
title: Fleet registrations must survive replacement of the app bundle
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-30T11:38:09Z'
updatedAt: '2026-09-30T12:24:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/345'
author: neo-gpt-emmy
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
closedAt: '2026-09-30T12:24:06Z'
---
# Fleet registrations must survive replacement of the app bundle

## Context

The operator's first installed FM registration succeeded on 2026-09-30. Preparing the subsequent app update revealed that `registry.json`, `credentials.enc`, and `fleet.key` had been written under `<app>/Contents/Resources/organism/.neo-ai-data/fleet`. The public registry contains the new seat; credential bytes were not inspected. The operator explicitly requires frequent FM updates without data loss.

## The Problem

The whole-app replacement procedure correctly preserves Electron userData, but Fleet's durable registry/key/ciphertext currently live inside the replaceable bundle. The launcher relocates working trees with `NEO_FLEET_AGENTS_ROOT` while leaving the separate durable Fleet store at its runtime-root default. The smoke's path census omits that store, so it can report no isolation violations while missing this write.

## The Architectural Reality

- `harness/brain.mjs:487` (`buildPackagedBrainEnv`) selects per-user writable paths but omits `NEO_FLEET_DATA_DIR`.
- `buildBrainProfile` has the same omission in checkout smoke.
- `resolveBrainPaths` projects `fleetAgentsRoot` but not `AiConfig.fleet.dataDir`; `assertIsolatedProfile` therefore cannot check the registry root.
- Brain `ai/configBase.mjs` already owns `fleet.dataDir` / `NEO_FLEET_DATA_DIR`. `FleetRegistryService.getDataDir()` consumes it for registry, ciphertext and key together.
- Existing owner: `harness/brain.mjs` launch profiles and their unit suite. No new module, service or config leaf. Brain structure-map was run during this onboarding lane; the Institution has no `ai:structure-map` host.

## The Fix

Set the existing Fleet durable-root environment leaf to `<dataRoot>/fleet` in the packaged profile and `<isolationRoot>/fleet` in checkout smoke. Include the resolved `fleetDataDir` in the runtime path report and isolation assertion. Extend the existing tests to fail when the store escapes or is absent, and verify the real Brain resolves it into the intended root.

Document upgrade recovery for already affected builds: before replacing an old bundle, preserve its entire Fleet store (registry, ciphertext and matching key) together and move it to the new durable location without overwriting a nonempty destination or exposing secret bytes. The actual operator migration is part of post-merge installation verification, not a claim that source tests performed it.

## Contract Ledger

| Surface | Authority | Behavior | Failure | Docs | Evidence |
|---|---|---|---|---|---|
| Packaged Fleet store | Existing Brain `fleet.dataDir` leaf; `buildPackagedBrainEnv` | User-data-owned root, independent of bundle location | Existing registry/key validation remains | `harness/README.md` updating section | Profile assertion and real resolver |
| Smoke Fleet store | `buildBrainProfile`, `resolveBrainPaths`, `assertIsolatedProfile` | Root is resolved and checked with the other mutable paths | Missing or escaping root fails isolation | Existing profile JSDoc | Positive and negative isolation controls |
| Existing installed data | Registry/ciphertext/key co-location contract | Retain complete store across whole-app replacement | Refuse blind overwrite of existing destination | Upgrade guidance | Post-merge operator receipt |

## Decision Record impact

Aligned with the existing explicit launch-profile/AiConfig leaf boundary; no AiConfig resolution changes. No new accepted decision is required for keeping mutable data outside a replaceable bundle.

## Acceptance Criteria

- [ ] AC-1: Packaged and checkout-smoke profiles set `NEO_FLEET_DATA_DIR` inside their explicit writable roots, independently of seat working-tree placement.
- [ ] AC-2: The real Brain path resolver reports `fleetDataDir`; isolation rejects missing, outside-root and symlink-escaping store paths.
- [ ] AC-3: Unit coverage exercises both profiles and a real Brain resolution; current-root defaults no longer silently satisfy the smoke path contract.
- [ ] AC-4: Upgrade guidance preserves an existing registry, ciphertext and matching key together, with no blind overwrite and no credential disclosure.

## Post-Merge Validation

On the currently affected installation, preserve the saved plane and complete Fleet store, replace the whole app, then confirm the existing seat is still listed and its credential resolves without re-entry. Continue the first-start witness under #12.

## Out of Scope

The fixture-plane smoke and userData isolation in #214 (Grace owns that lane), Chroma startup, automated release distribution, seat-directory migration, provider authentication and wake setup.

## Related

#12 · #214 · #7 · neomjs/neo-agent-brain#571

Sweeps: live latest 25 open Institution issues, own assignments (none), and recent all-state A2A reviewed immediately before filing; no duplicate persistence repair. #214 body and active ownership checked; separate scope sent to Grace. MC query `packaged fleet registry saved inside app bundle data root` found prior empty-registry and update receipts, not a prior fix. The first nonempty registry makes this update defect observable. Prior empty-registry receipt: memory `39d81df8-3027-43c0-bc3d-fac2a30aebcc`.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e.
Retrieval Hint: packaged Fleet registry dataDir app replacement credentials.enc fleet.key.

## Timeline

- 2026-09-30T11:38:10Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-30T11:38:11Z @neo-gpt-emmy added the `bug` label
- 2026-09-30T11:38:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-09-30T11:38:12Z @neo-gpt-emmy added the `ai` label
- 2026-09-30T11:42:32Z @neo-opus-grace cross-referenced by #214
- 2026-09-30T11:45:07Z @neo-gpt-emmy cross-referenced by PR #346
- 2026-09-30T11:47:02Z @neo-fable-clio cross-referenced by #335
- 2026-09-30T11:56:50Z @neo-fable-clio cross-referenced by #13
- 2026-09-30T12:02:06Z @tobiu referenced in commit `4fcc0fd` - "Merge pull request #346 from neomjs/codex/345-fleet-durable-root

fix(harness): keep Fleet state outside the app bundle (#345)"
- 2026-09-30T12:13:55Z @neo-opus-vega cross-referenced by #347
### @neo-gpt-emmy - 2026-09-30T12:24:01Z

Installed persistence witness completed on 2026-09-30 using Institution `4fcc0fd`, Brain `8a42800`, Engine `067f9fb` and Electron 43.5.0.

- Quit the old shell; retained its bundle and a profile backup.
- Preserved the complete Fleet store under `<userData>/brain/fleet`: registry, ciphertext and matching key copied byte-for-byte, retaining `0600` file permissions and the existing seat workspace.
- Installed the whole replacement app from the clean build ZIP. The packaged shell module matched the merged source; the archive contained no runtime-root `.neo-ai-data` store.
- Reopened into the same saved plane and viewer identity. The existing `neo-gpt-sophie` row survived, including its enabled GitHub Workflow override.
- Resolved the migrated encrypted credential through Fleet and made a read-only GitHub identity request: the authenticated login is `neo-gpt-sophie`, with no credential re-entry or secret output.

This completes the Fleet persistence slice. The new peer's first session remains an independent onboarding witness under #12. The broader plane-member placement class is now owned by Vega in #347. Standalone smoke still reports `chromaListening: false`; renderer, popup/IPC and clean teardown pass, so no full standalone-smoke success is claimed.

Origin Session ID: b0dd802b-6451-48ec-b789-d91e29a2b08e.

- 2026-09-30T12:24:07Z @neo-gpt-emmy closed this issue
- 2026-09-30T13:33:16Z @neo-gpt cross-referenced by PR #348

