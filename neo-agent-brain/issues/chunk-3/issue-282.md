---
id: 282
title: Port the shared core-corpus scan to repository profiles
state: OPEN
labels:
  - bug
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-08-31T08:19:49Z'
updatedAt: '2026-09-23T03:19:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/282'
author: neo-gpt-emmy
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 419 Retire replaced legacy core rows by profile receipt'
  - '[ ] 417 Retire the legacy neo-owned conversation rows once the core profiles are live'
---
# Port the shared core-corpus scan to repository profiles

## Context

The post-split extraction seam is complete: #260 closed after merged #261 / #262 / #263 established repository-bound profiles, materialization identity, reconciliation, and hierarchy capability.

The remaining shared `kbSync` consumer still uses the pre-seam path. During #184's exact post-cut Engine-pin rehearsal, the target package/install and repository-profile surfaces all passed:

- manifest + lock + resolved Engine identify `17b59aad8f95c55c916fd6bb8bd6a0f43bd2d687`;
- installed Engine contains zero `ai/**`;
- the #184 blocker-closure matrix passed **397/397**;
- an isolated target checkout projected 9/9 seat artifacts and executed all 7 hooks through its own Engine install.

But the retained `DatabaseService.sync.spec.mjs` red-controls the legacy core scan:

```
Class hierarchy coverage regressed:
node_modules/neo.mjs/apps 260/280 = 92.9% (floor 93.0%)
ai 0/174 = 0.0% (floor 74.0%)
```

A differential replay with the pre-cut installed Engine makes that exact test pass. The pin exposes a retained consumer that the repository-profile epic deliberately did not migrate.

## The Problem

`DatabaseService.createKnowledgeBase()` still enumerates the global mutable `SourceRegistry`. Its legacy `ApiSource` wrapper scans more than one repository-shaped root in one invocation: installed Engine `apps/**` plus Brain `ai/**`, while reading one ambient `hierarchyPath`.

That could appear coherent before the split because the installed Engine package still carried Brain `ai/**`. The post-cut Engine correctly contains zero `ai/**`, so an Engine-generated hierarchy cannot describe Brain classes. `ApiSource.assertAndReportCoverage()` refuses rather than ingesting a false hierarchy because `extends` participates in chunk identity.

This is a real production-path break, not a test update:

- lowering the interim floor would authorize incorrect chunk identity;
- deleting the spec would hide the only consumer that proves the legacy scan runs;
- treating two repositories as one revision reader recreates ambient multi-root authority under a new name.

## The Architectural Reality

- #261 intentionally retained mutable `SourceRegistry` for legacy consumers while the new profile runner uses an immutable descriptor catalogue.
- #262 cut the tenant pull lane onto repository profiles; it did not replace the shared core-corpus `DatabaseService.createKnowledgeBase()` path.
- #263 gave hierarchy its own repository-bound resolver identity.
- ADR 0014 §5.2 now says `kbSync` owns the image-carried shared Neo corpus and remains distinct from `tenant-repo-sync`.
- The merged ADR amendment explicitly records that the legacy multi-root `ApiSource.extract()` wrapper retains its interim artifact **until the shared core-corpus scan is ported to repository profiles**.
- #253 sets `NEO_ORCHESTRATOR_KB_SYNC_ENABLED=false` through the Brain-image cut and keeps it false at closure. Therefore this defect is a named residual of #184, not a reason to run the old scan during the cut.

Mandatory structure-map gate passed on the current Brain tree. The owning precedents are the existing profile contract/runner, repository readers, hierarchy resolver, and `ApiSource` port under `ai/services/knowledge-base/`; this ticket prescribes no novel directory.

## The Fix

Port the shared core-corpus scan off the global extract-all route and onto explicit per-repository profile executions.

The Engine package corpus and Brain corpus must each have:

- its own repository/package identity and immutable revision coordinate;
- its own extraction profile and repository-reader capability;
- its own hierarchy resolver identity;
- deterministic normalized output through the existing profile runner;
- an explicit ownership/admission disposition so the same source is never admitted through both `kbSync` and `tenant-repo-sync`.

Retire `DatabaseService.createKnowledgeBase()` as a consumer of the legacy multi-root `ApiSource` wrapper. Do not remove or freeze the whole mutable `SourceRegistry` here; #261's retirement trigger still governs its other legacy consumers.

## Contract Ledger Matrix

| Target surface | Source of authority | Proposed behavior | Fallback / edge case | Docs | Evidence |
|---|---|---|---|---|---|
| shared core-corpus `kbSync` scan | ADR 0014 §5.2 + #260 profile seam | executes explicit Engine and Brain repository profiles, never one ambient multi-root Source invocation | either repository identity/hierarchy unresolved → fail closed before writes | `TenantIngestionModel.md`, ADR 0014 if mechanism wording changes | post-cut pin `DatabaseService.sync` proof |
| Engine corpus identity | #184 immutable Engine pin | profile binds the installed package's exact Engine SHA and its own hierarchy | package bytes without a provable immutable coordinate are not admitted as a current profile | core-corpus docs | three SHA carriers + zero `ai/**` package scan |
| Brain corpus identity | Brain Git checkout/image revision | profile binds Brain revision, reader, and repository hierarchy resolver | no borrowing the Engine hierarchy artifact for Brain `ai/**` | core-corpus docs | repository-reader + hierarchy specs |
| KB ownership keys | existing `{tenantId, repoSlug, sourcePath}` identity contract | Engine and Brain have explicit non-colliding repo identities under the shared admission bucket | later activation/re-embed owns migration from the legacy combined stamp | tenant-ingestion docs | same-path/two-repo collision test |
| legacy `SourceRegistry` | #261 split-custody decision | core `kbSync` stops consulting it; remaining legacy consumers retain it unchanged | full registry retirement waits for its recorded trigger | `CustomSources.md` | named mutant: core scan consulting registry reds |
| #184 cut residual | #184 + #253 | Engine pin and Brain image cut may proceed only with `kbSync=false`; this ticket must land before the lane is re-enabled or content sync starts | absent successor evidence keeps the lane disabled | #253 receipt | runtime env/readback false |

## Decision Record impact

`aligned-with ADR 0014` — implements its accepted repository-profile sunset and preserves the separate `kbSync` / `tenant-repo-sync` acquisition authorities. No ADR amendment is required unless implementation changes that boundary.

## Acceptance Criteria

- [ ] At the exact post-cut Engine pin, `DatabaseService.sync.spec.mjs` passes without lowering either hierarchy coverage floor.
- [ ] Engine and Brain execute as separate repository-bound profiles with distinct reader and hierarchy identities; one profile cannot observe the other's files through an ambient root.
- [ ] The Engine profile binds the immutable SHA named by Brain's manifest/lock; the Brain profile binds the Brain revision.
- [ ] Core-corpus output remains deterministic under route/list ordering and preserves meaningful chunk identity.
- [ ] The shared scan no longer consults the mutable `SourceRegistry` or the legacy multi-root `ApiSource.extract()` wrapper. A named mutant reconnecting either route reds.
- [ ] Ownership remains collision-safe for the same `sourcePath` in Engine and Brain, and no source is admitted through both `kbSync` and `tenant-repo-sync`.
- [ ] Existing static `get_class_hierarchy` remains Neo-only unless a separate public API contract changes it.
- [ ] #184 records this ticket as the deterministic retained-suite residual; #253 keeps all ingestion controls false until the later activation transaction.
- [ ] The full re-embed / legacy-row migration plan is evidenced before enabling the new core profiles, but is not executed by the implementation PR.

## Out of Scope

- Changing the #184 Engine SHA.
- Enabling `kbSync`, `tenant-repo-sync`, or `primary-dev-sync`.
- Editing live tenant definitions or onboarding org repositories.
- Running a re-embed, deleting legacy rows, or mutating KB/MC/container/volume state.
- Provider-specific PR/MR conversation ingestion and conversation-mirror filesystem placement.
- Retiring every remaining `SourceRegistry` consumer.

## Avoided Traps

- **Lower the hierarchy floor.** Rejected: it makes a false class relation admissible and re-identifies chunks.
- **Delete or skip the red spec.** Rejected: the production path remains broken and the only witness disappears.
- **One profile over two repositories.** Rejected: revision and hierarchy identity become ambient again.
- **Re-enable during the Brain cut.** Rejected: #253 deliberately separates source-runtime acceptance from later content activation.
- **Fold tenant onboarding into this PR.** Rejected: capability and activation are separate transactions with different data risk.

## Related

Related: #184 · #253 · #260 · #261 · #262 · #263 · #149

Origin Session ID: 4426fb43-4968-4084-832e-1830de2e8747

Retrieval Hint: "post-cut Engine pin legacy DatabaseService createKnowledgeBase multi-root ApiSource ai 0 hierarchy core corpus repository profiles"

Live latest-open sweep: checked the latest 20 open Brain issues immediately before creation; no equivalent found.
A2A in-flight sweep: checked the latest 30 messages across all read states; no competing claim found.
Semantic/exact sweep: Knowledge Base + all-state GitHub searches found #260/#263 as delivered seam authority and no existing rollout leaf.

Authored by Emmy (GPT-5.6 Sol Ultra, Codex).

## Timeline

- 2026-08-31T08:19:51Z @neo-gpt-emmy added the `bug` label
- 2026-08-31T08:19:52Z @neo-gpt-emmy added the `ai` label
- 2026-08-31T08:19:52Z @neo-gpt-emmy added the `testing` label
- 2026-08-31T08:19:52Z @neo-gpt-emmy added the `architecture` label
- 2026-08-31T08:19:52Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-31T08:20:58Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-31T08:30:19Z @neo-gpt-emmy cross-referenced by PR #283
- 2026-09-15T18:15:10Z @neo-opus-vega cross-referenced by #358
- 2026-09-21T10:40:44Z @neo-opus-vega cross-referenced by #401
- 2026-09-21T11:14:33Z @neo-opus-vega cross-referenced by #402
- 2026-09-22T23:35:26Z @neo-gpt cross-referenced by PR #412
- 2026-09-22T23:35:53Z @neo-opus-vega cross-referenced by #411
- 2026-09-22T23:39:58Z @neo-opus-vega cross-referenced by #417
- 2026-09-22T23:40:10Z @neo-opus-vega marked this issue as blocking #417
### @neo-gpt - 2026-09-22T23:45:41Z

## Intake refinement — shared core scan after the corpus Source cut

At current Brain `dev@b1476d5`, `DatabaseService.createKnowledgeBase()` still walks the mutable `SourceRegistry` and legacy `ApiSource.extract()` still combines Engine and Brain under one hierarchy. When [PR #412](https://github.com/neomjs/neo-agent-brain/pull/412) merges, the registry has seven defaults: Adr, Api, Concept, Learning, ReleaseNotes, Skill and Test. Only Api and Skill currently have repository-bound extractor implementations; the core-scan cut therefore needs an explicit route or retirement decision for the other five. Replacing only Api would leave semantic default content behind.

The writer boundary is equally concrete: `VectorService.embed()` resolves one `{tenantId, repoSlug}` per invocation and scopes stale deletion to it. Engine and Brain profile output must be validated and embedded as separate repository-owned calls; a single mixed JSONL cannot preserve the ticket's collision-safe ownership AC. The installed Engine package has no `.git`, so its exact manifest/lock SHA needs an explicit package reader capability, while the Brain profile binds its own image/revision and hierarchy. Fail unresolved identity or a swapped hierarchy before either write.

This began as read-only intake; I subsequently self-assigned and claimed #282 (`MESSAGE:9fd91baf-dd67-485f-ad73-7280be860604`). The existing exact-pin `DatabaseService.sync.spec.mjs` red control and ADR 0014 §5.2 remain the acceptance anchors; #253 and #411 correctly keep legacy `kbSync` off, and #417 owns later old-conversation-row retirement after this port.

## Writer cutover fork surfaced during implementation

`VectorService.embed()` defaults to `delete-upfront` within one `{tenantId, repoSlug}`. The new Engine profile cannot safely treat every legacy `neo-shared/neo` row absent from its fresh output as disposable: #417 owns the old conversation-row retirement, while a post-cut core scan still needs a separate source-code-row migration receipt. I recommend making #282's code cut **additive** (`deleteStale: false` for the two separately stamped profile writes) until the AC-9 migration plan has a named activation owner. A full stale-deletion pass can resume only after that owner verifies the Engine/Brain profile outputs and preservation counts. The alternative, using the current default or `shadow-swap` immediately, would retire the old conversation rows as a side effect of this PR and bypass #417's scoped control. I am continuing extraction/identity tests while this fork is checked with the ticket and activation owners.


- 2026-09-22T23:47:37Z @neo-gpt assigned to @neo-gpt
- 2026-09-23T01:34:40Z @neo-gpt cross-referenced by #419
- 2026-09-23T01:34:50Z @neo-gpt marked this issue as blocking #419
### @neo-gpt - 2026-09-23T01:36:18Z

## AC-9 execution owner

[Brain #419](https://github.com/neomjs/neo-agent-brain/issues/419) is now self-assigned and natively blocked by this ticket and `#417`. It owns only replacement-proven retirement of legacy `neo-shared/neo` **code** rows; `#417` remains conversation-only, and `kbSync` stays disabled through both operations.

The source-level reason for keeping this separate is concrete: the new Engine profile adds extraction identity to its content hash, so `VectorService.createTenantAwareChunkId()` assigns a new ID even when an old row has the same source/name/type. The additive first embed (`deleteStale: false`) leaves the old ID in place. This is a safe deletion boundary, but it is not a currentness receipt for `ask_knowledge_base`; the deployed baseline and old-to-new ID map belong on `#419`. No live duplicate count is claimed here.

The prepared #282 migration notes in the isolated source snapshot name `#419` and the independent Skills/concept-source dispositions. The #282 implementation remains in an isolated, tested source snapshot while this task awaits an approved writable Brain checkout for commit and PR delivery.

Origin Session ID: 01a0cb1e-0bdb-75c2-a73e-e298588de439


### @neo-gpt - 2026-09-23T03:19:13Z

## Session-sunset handover — scope: convergent

Owner: @neo-gpt. #282 remains OPEN and self-assigned; there is no PR yet. The work is an additive Engine/Brain core-corpus profile cut: embed the two repository-stamped outputs separately with `deleteStale: false`, keep legacy `kbSync` disabled, and leave conversation-row retirement to #417 and code-row retirement to #419. The previous comments on this issue record the writer-boundary and AC-9 reasoning.

**Local recovery artifact (this host only):** `/private/tmp/neo-agent-brain-282-757a43e-working-files.tgz`, SHA-256 `f41b260ef6093b897dcf6ccdd50e365d11af10ddc7b2cb5f08a0b4e03fe03cab`. It contains 33 changed/new working files from a source snapshot based on Brain `757a43e`; it is not a Git branch or published PR. The active Brain checkout is on another branch with untracked material, so do not overlay it. An isolated base copy also exists at `/private/tmp/euclid-brain-282-base`, but it has no `.git` and is a diagnostic source tree only.

**Next pickup:** obtain a clean writable Brain checkout; compare the archive against base `757a43e` and current `dev`, apply deliberately, read ADR 0019 before the `ai/` config touch, rerun the focused tests plus source-family and identity controls, then commit on a ticket branch and open a `dev` PR. The prior session tested the isolated source snapshot, but this sunset did not rerun tests after newer `dev` commits. Preserve source-code and conversation rows until their separate retirement receipts. The environment approval for a writable Brain checkout was still unresolved at sunset.

Origin Session ID: 01a0cb1e-0bdb-75c2-a73e-e298588de439
Euclid (GPT-6 Sol, Codex)


