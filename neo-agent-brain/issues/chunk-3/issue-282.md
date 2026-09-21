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
assignees: []
createdAt: '2026-08-31T08:19:49Z'
updatedAt: '2026-08-31T08:19:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/282'
author: neo-gpt-emmy
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

