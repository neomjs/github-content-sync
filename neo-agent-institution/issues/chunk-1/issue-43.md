---
id: 43
title: Consume Brain Fleet contract and delete vocabulary twins
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - architecture
  - dependencies
  - refactoring
  - testing
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-28T22:34:34Z'
updatedAt: '2026-09-05T15:13:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/43'
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
blockedBy:
  - '[x] 330 Expose tenant-target support in the public harness catalog'
  - '[x] 217 Expose one client-safe Fleet contract from Brain'
blocking: []
closedAt: '2026-09-05T15:08:22Z'
---
# Consume Brain Fleet contract and delete vocabulary twins

## Problem

Agent Institution currently duplicates Brain-owned Fleet vocabulary in four local modules. The parity test that was meant to protect those twins now skips post-split. Source parity therefore does not protect the duplicated definitions.

Current production consumers include the harness pickers, `AgentConfigComponent` (MCP catalog and tenant capability), `SourceHealth` (cockpit source identifiers), and `installFleetBridge` (wire helpers). They need the installed client contract rather than local vocabulary twins or a Brain checkout root.

## Scope

After Brain #217 exports the dependency-free `neo-agent-brain/fleet-contract` subpath, pin that exact Brain revision in Agent Institution and import the contract from the installed package.

Update the real view, bridge, harness, and test consumers. Delete all four Institution vocabulary twins and the source-parity spec. Retain protocol-skew behavior tests: compatibility and fail-closed negotiation are product behavior, unlike source equality.

## Contract Ledger

| Surface | Authority | Delivered behavior | Boundary / fallback | Evidence |
|---|---|---|---|---|
| Package and lockfile | Brain #217 / merged PR #326 plus capability completion #330 / PR #331 | Immutable installed Brain dependency | No floating producer ref | Normal install and lock read-back |
| Browser/App-Worker imports | Brain `package.json.exports["./fleet-contract"]` → `src/fleet/contract/index.mjs` | Views and bridge import that public installed entry by the existing relative dependency-URL convention | Unbundled workers have no bare-specifier resolver; no copied vocabulary, wrapper or private import | Real development boot and production browser build; export-target/path agreement |
| Node wire consumers | `neo-agent-brain/fleet-contract` | Tests import the public package subpath; the packaged launcher resolves its installed export target from explicit `productRoot` | No retired `ai/services/fleet/fleetWireMethods.mjs` import | Launcher and protocol-skew tests |
| Trusted launcher classification | Existing private `fleetLaunchContract.mjs` on explicit runtime root | Read `FLEET_CREDENTIAL_METHODS` alongside the existing bearer/probe seams | Private runtime root remains separate from client package and target checkout; never imported by App Worker | Actual launcher/capability composition |
| Four local twins and parity spec | Installed public contract | Delete duplicated definitions and skipped source-equality test | Preserve behavior/skew tests; no replacement parity/sync surface | Import census and existing rejection controls |
| Cross-repository CI | Same immutable Brain revision as package/lock | Provision the private test runtime at that revision | Keep one physical Engine identity in that tier; collection is not execution | CI unit execution and explicit E2E receipts |
| Packaged source-mode assets | `harness/contentPolicy.mjs` + `harness/pack.mjs` | Admit and stage only the pinned public Fleet contract graph alongside the installed package | No broad Brain/private asset access; no vocabulary copy maintained as source | Packaged-layout import and content-policy negative controls |

Decision Record impact: aligned-with ADR 0038 §2.8 (wire-only client contract) and ADR 0040 §2.5 (separate runtime and target roots).

## Acceptance Criteria

- [ ] `package.json` and the lockfile pin the immutable Brain producer revision that exports `neo-agent-brain/fleet-contract`.
- [ ] `AddAgentForm`, the Accounts panel, and `installFleetBridge` consume the installed public subpath rather than local twins or a runtime-root filesystem import.
- [ ] `apps/agentos/config/harnessTypes.mjs`, `mcpServers.mjs`, `fleetWireMethods.mjs`, and `cockpitSources.mjs` are deleted with all stale imports/comments.
- [ ] The skipped `fleetVocabularyParity.spec.mjs` is deleted; no replacement parity lint, sync, generated copy, or env-injected twin root is added.
- [ ] Tests that exercise wire-version/capability skew remain and still fail closed against incompatible responses.
- [ ] A fresh isolated install and browser build resolve the contract without `NEO_AGENTOS_RUNTIME_ROOT`, sibling checkouts, Brain service state, generated Brain configuration, or seat hook artifacts. Existing Skills materialization during normal installation is not a claim of zero lifecycle execution.
- [ ] The App-Worker bundle pulls no Node builtin, Brain service implementation, credential policy, or second `neo.mjs` class identity through the contract.
- [ ] Unit and e2e suites remain green on the exact consumer pin.

## Out of scope

- redesigning the Fleet wire protocol;
- exposing Brain-private credential or lifecycle policy;
- adding a dedicated contracts package unless #217's named package/bundling falsifiers prove the existing subpath impossible.

## Relationships

Producer neomjs/neo-agent-brain#217 is closed. Capability producer neomjs/neo-agent-brain#330 is also closed; PR #118 delivers the adoption. Consumer-side corrective successor to neomjs/neo-agent-brain#206.

## Coordinated landing

Brain PR #326 merged at `a9e010d75420ecb71e7d6a8553af1be5aba8bc52`. Institution PR #116 then repaired the immediate runtime-path break and merged at `1fb92af71241d1907ef07569f0668259583242e9`; this adoption builds on that repair.

Brain PR #331 merged as `bd5417155be455b59983de2a6e5bd8249b113d2b`. PR #118 pins that merged revision in package, lockfile and the explicit runtime CI ref; it is ready and cross-family approved at `c2b3ab907392e51712e8e8a0a9da019b94e34616`. Human merge remains the final gate.

The trusted Electron launcher resolves public vocabulary from the installed package under explicit `productRoot`; identity, credential classification and authenticated probe primitives remain under the separately selected `runtimeRoot`. The packaged organism explicitly supplies the same root for both roles. The browser imports only the public export target.

Project `.npmrc` sets `foreground-scripts=true`, retaining ordinary lifecycle execution while serializing the overlapping Engine/Brain Skills materializers for every normal install, including the reusable baseline's plain `npm ci`. The separately provisioned Brain CI checkout uses the explicit flag. The parallel default reproduced an EEXIST race on the shared projection link.

## Capability completion

The adoption sweep found that `AgentConfigComponent` also consumes the tenant-target support predicate. Merged Brain #217 had classified it private; Brain #330 / PR #331 completes the public harness catalog with that configuration capability while normalization and credentials remain private. The final consumer pin must include that merged completion. Prepare against its immutable candidate for review; the native dependency is the final landing gate. No copy of the private support list or private worker import is allowed.


## Timeline

- 2026-08-28T22:34:35Z @neo-gpt-emmy added the `bug` label
- 2026-08-28T22:34:35Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T22:34:36Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T22:34:36Z @neo-gpt-emmy added the `architecture` label
- 2026-08-28T22:34:36Z @neo-gpt-emmy added the `dependencies` label
- 2026-08-28T22:34:36Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-28T22:34:37Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T22:35:06Z @neo-gpt-emmy cross-referenced by #206
- 2026-08-30T18:51:19Z @neo-opus-ada cross-referenced by PR #255
- 2026-08-30T21:12:53Z @neo-gpt-emmy cross-referenced by #64
- 2026-09-05T01:19:57Z @neo-gpt-emmy cross-referenced by PR #326
- 2026-09-05T12:18:51Z @neo-fable-clio cross-referenced by PR #114
- 2026-09-05T12:32:10Z @neo-fable-clio cross-referenced by #115
- 2026-09-05T12:35:29Z @neo-fable-clio cross-referenced by PR #116
### @neo-gpt-emmy - 2026-09-05T12:36:27Z

## Intake — valid-as-written after contract alignment

Self-authored 2026-08-28; the drift probe intersects the wire twin, launcher, package/lock and CI, so the full intake ran. Current `dev@ff70c15` still carries the four twins and old-path harness imports; no merged adoption or competing PR was found. The only open PR, `#114`, overlaps the wire import seam; Clio has the exact handoff. Brain `#217` is closed through merged `#326` (`a9e010d75420ecb71e7d6a8553af1be5aba8bc52`). No parent epic or stale/close-inactive workflow exists on this repository; no stale/not-code-ready labels are present.

The missing Contract Ledger is now in the body. Source-mode SharedWorkers consume the installed public export target by URL, following the existing dependency-path precedent; Node uses `neo-agent-brain/fleet-contract`. Packaging must stage/admit that same public graph, not all Brain assets. The trusted launcher keeps its explicit runtime root and reads the private credential-method classification from `fleetLaunchContract.mjs`. These are transport/ownership dispositions for the existing client contract, not a new contract package or policy copy.

ADR successor-risk: **adr-aligned** — ADR 0038 §2.8 keeps the client import graph free of trust policy; ADR 0040 §2.5 preserves separate runtime/target roots. No ADR amendment or new runtime module is prescribed. Existing source-class behavior stays unchanged; the view edits replace imports only.

Memory/KB instruments remain partly degraded: semantic recall surfaced the preceding producer-repair turn `fa99ced9-1a02-4677-b8b2-d9c2735d47b1` alongside irrelevant records; KB synthesis could not load its configured model, and pre-brief found no repo-qualified issue node. Live producer/consumer source, native dependency and reviewed handoff supply the readiness evidence. ROI is positive: remove four drift-prone definitions and the skipped parity spec while repairing the now-live launcher/CI import break.

Origin Session ID: 0e9cccb4-3c5c-4f04-be91-badb29c50238

- 2026-09-05T12:37:16Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-05T13:05:19Z @neo-gpt-emmy cross-referenced by #330
- 2026-09-05T13:19:15Z @neo-gpt-emmy cross-referenced by PR #331
- 2026-09-05T14:06:28Z @neo-opus-ada cross-referenced by PR #118
- 2026-09-05T14:13:25Z @neo-opus-ada referenced in commit `502cb5c` - "chore(fleet): keep touched comments timeless (#43)"
- 2026-09-05T14:28:44Z @neo-opus-ada referenced in commit `33338bc` - "chore(deps): pin the merged Fleet producer (#43)"
- 2026-09-05T14:36:45Z @neo-opus-ada referenced in commit `c2b3ab9` - "chore(deps): serialize shared install lifecycle scripts (#43)"
- 2026-09-05T15:08:22Z @tobiu referenced in commit `285446d` - "Merge pull request #118 from neomjs/codex/43-canonical-fleet-contract

fix(fleet): consume the installed public contract (#43)"
- 2026-09-05T15:08:22Z @tobiu closed this issue

