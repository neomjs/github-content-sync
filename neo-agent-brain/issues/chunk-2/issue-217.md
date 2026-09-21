---
id: 217
title: Expose one client-safe Fleet contract from Brain
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
  - testing
  - architecture
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-28T22:33:54Z'
updatedAt: '2026-09-05T12:10:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/217'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 193
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 184 Align Brain''s Engine pin with post-split consumers'
blocking:
  - '[x] 43 Consume Brain Fleet contract and delete vocabulary twins'
closedAt: '2026-09-05T12:10:48Z'
---
# Expose one client-safe Fleet contract from Brain

## Problem

Brain and Agent Institution still maintain separate Fleet vocabularies. The remaining producer work is a single dependency-free client contract exported by the existing Brain package; Institution #43 owns adoption and deletion of its twins.

The old roster-import and dead-lint findings have already been repaired. Intake at Brain `442a220` verified that `deriveFleetRoster.mjs` imports Brain's cockpit source authority, and the retired parity lint/script/spec are absent. This ticket preserves that result rather than reintroducing or re-fixing it.

## Scope

Expose `neo-agent-brain/fleet-contract`, backed by canonical `src/fleet/contract/**` under #193.

Move the existing pure harness catalog/helpers, MCP catalog/default/sparse-override helpers, Fleet wire method/version/capability/envelope helpers, and cockpit source/event identifiers into this authority. Update Brain production/test imports and delete the displaced public definitions; no compatibility re-export at the old paths.

MCP credential environment names, target normalization/capability policy, bearer/identity resolution, authorization, process environment, filesystem access, stores and lifecycle services remain private. The public import must reach none of them.

## Contract Ledger

| Surface | Authority | Behavior | Fallback / boundary | Evidence |
|---|---|---|---|---|
| `neo-agent-brain/fleet-contract` | #193 canonical domains; ADR 0038 client topology | Named package exports from `src/fleet/contract/index.mjs` | No Node/service/config/credential import closure; no new package | Installed-package import and browser bundle |
| Harness catalog | Current `harnessTypes.mjs` | Frozen keys/labels; caller-owned lists and lookup records | Unknown harness resolves null | Existing harness behavior plus public-entry tests |
| MCP catalog and matrices | Current public portion of `mcpServers.mjs` | Catalog, defaults, resolve/normalize sparse overrides | Unknown keys and non-boolean writes fail closed; credential/target policy stays private | Existing matrix tests and private-export negative control |
| Fleet wire | Current `fleetWireMethods.mjs` | Same verbs, protocol/capability negotiation, request/response helpers and closed states | Unknown method/protocol/capability still fails closed; no trust-policy exports | Existing wire/dispatch tests plus installed-entry probes |
| Launcher credential-method classification | Existing Fleet launcher trust boundary | `FLEET_CREDENTIAL_METHODS` remains frozen in private `ai/services/fleet/fleetLaunchContract.mjs`, derived from the canonical wire verbs | Not a browser/package export; Institution's trusted runtime loader consumes this private home | Actual Institution `createFleetCapability` rejects absence and accepts the restored classification |
| Cockpit identifiers | Current `fleetCockpitStatus.mjs` declarations | One source/event vocabulary shared by Brain DTO producers and clients | DTO assembly/redaction/lifecycle remains private | Producer tests and canonical imports |
| Package consumption | Existing Brain package | Consumer can install and import the client subpath without service-state or lifecycle materialization | Root development setup remains explicit and verified; no consumer checkout-root injection | Normal isolated consumer install and filesystem/import checks |
| Architecture map | Engine `learn/benefits/ArchitectureOverview.md` | Inventory names Brain's new Fleet contract home and client/private boundary | Linked companion before producer PR is declared ready | Companion diff and reference checks |

## Acceptance Criteria

- [ ] The documented package subpath resolves to canonical `src/fleet/contract/**`.
- [ ] Its static/transitive imports contain no Node builtin, `neo.mjs` class, service implementation, credential/target policy, store singleton, filesystem or environment access.
- [ ] Existing harness/MCP/wire behavior is preserved, including malformed/unknown-value rejection and protocol-version negotiation.
- [ ] Brain production consumers use the canonical contract; old public definitions are deleted, with no forwarding aliases or twins.
- [ ] Private credential and target helpers are absent from public exports and the reachable source graph.
- [ ] Focused tests exercise the installed public entry; an isolated browser bundle reaches only the contract.
- [ ] A normal isolated consumer install/import creates no Brain service state or lifecycle artifacts; developer/bootstrap behavior remains verified.
- [ ] The earlier roster-import and dead parity-lint repairs stay intact.
- [ ] The Engine inventory companion and Brain package usage documentation describe the final boundary.

## Out of scope

Fleet protocol redesign; Institution UI changes or its pin/adoption (#43); deployment, credential or authorization changes; a new contracts package without a named falsifier of the existing package boundary.

## Intake and authority

Created 2026-08-28; revalidated 2026-09-05. The self-authored drift probe intersects the Fleet sources and package manifest, so full intake was applied. #184 is closed; no open native blocker or competing producer PR/claim was found. Parent #193 is self-authored and already prescribes the canonical domain root; `src/evolution` is its landed source precedent.

Classification: valid-as-written after this author-side drift restatement. ADR successor-risk: aligned with accepted ADR 0038's pure-client boundary and ADR 0040's two-root/dependency direction; no ADR amendment. The KB/prior-art queries did not recover a useful #217 graph node; live sources and issue authority decide the scope.

Origin Session ID: 4b8faa04-faf0-49ef-86a4-c77e9cb0a621

## Timeline

- 2026-08-28T22:33:55Z @neo-gpt-emmy added the `bug` label
- 2026-08-28T22:33:56Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T22:33:56Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-28T22:33:56Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T22:33:56Z @neo-gpt-emmy added the `architecture` label
- 2026-08-28T22:33:56Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T22:34:35Z @neo-gpt-emmy cross-referenced by #43
- 2026-08-28T22:35:06Z @neo-gpt-emmy cross-referenced by #206
- 2026-08-28T22:48:58Z @tobiu cross-referenced by #184
- 2026-08-30T17:45:56Z @neo-gpt-emmy cross-referenced by PR #255
- 2026-08-30T18:54:27Z @neo-gpt-emmy cross-referenced by #198
- 2026-08-30T21:12:53Z @neo-gpt-emmy cross-referenced by #64
- 2026-09-04T20:27:13Z @neo-fable-clio cross-referenced by #314
- 2026-09-05T00:23:35Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-05T00:54:59Z @neo-gpt-emmy cross-referenced by #18349
- 2026-09-05T01:09:44Z @neo-gpt-emmy cross-referenced by PR #18350
- 2026-09-05T01:19:57Z @neo-gpt-emmy cross-referenced by PR #326
- 2026-09-05T01:23:11Z @neo-gpt-emmy referenced in commit `a4b1a4e` - "docs(fleet): follow the canonical wire contract path (#217)"
- 2026-09-05T11:47:53Z @neo-gpt-emmy referenced in commit `a16d5f2` - "fix(fleet): retain private launcher credential methods (#217)"
- 2026-09-05T12:10:48Z @tobiu referenced in commit `a9e010d` - "Merge pull request #326 from neomjs/codex/217-client-safe-fleet-contract

feat(fleet): export the client-safe Fleet contract (#217)"
- 2026-09-05T12:10:48Z @tobiu closed this issue
- 2026-09-05T12:32:10Z @neo-fable-clio cross-referenced by #115
- 2026-09-05T12:35:29Z @neo-fable-clio cross-referenced by PR #116
- 2026-09-05T13:05:19Z @neo-gpt-emmy cross-referenced by #330
- 2026-09-05T13:19:15Z @neo-gpt-emmy cross-referenced by PR #331
- 2026-09-05T15:00:25Z @neo-fable-clio cross-referenced by PR #118

