---
id: 1329
title: Qualify community coverage without promoting legacy receipts
state: OPEN
labels:
  - enhancement
  - ai
  - testing
  - architecture
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-28T00:08:33Z'
updatedAt: '2026-10-28T00:08:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1329'
author: neo-gpt-euclid
commentsCount: 0
parentIssue: 106
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
# Qualify community coverage without promoting legacy receipts

## Context

The coverage amendment in ADR 0036 is now human-merged through #1328 at `d02bdd77c0400a29411b9846ec37959ded08d0f7`, following [D20161](https://github.com/orgs/neomjs/discussions/20161). Parent #106 still owns the real tenant/source/store-bound shadow window. This leaf implements the accepted qualification boundary through the existing producer → admission → public-count → shared-evaluator path.

Design authority: ADR 0036 §2.4: “A legacy complete-looking receipt without its declaration binding, an unknown version or mixed family generations cannot qualify a source.” §2.6 requires all three GitHub repository families and preserves existing actor/closer predicates.

## The Problem

At source `3bc4a9a91de94f0666b010d32ad5db60eb633480`:

- PR and Discussion runners place fixed provider-history limitations into `coverage.gaps`, permanently degrading those families even after supported acquisition succeeds.
- `CommunityActivityService.readCoverage` recognizes only `complete:true` plus an empty gap array. It has no persisted declaration/generation qualification.
- `assembleIssueBatch` rebuilds a partial coverage object and drops additional evidence.
- The hosted gap item accepts only optional `fromBasis/toBasis` fields, refusing actual producer `axis/reason`, cursor, entity and count fields. The neutral envelope validator accepts those actual producer shapes.
- The shared evaluator returns early for empty demand before observing lease vacancy.

This prevents a valid retained window and also risks accepting legacy complete-looking evidence without the new binding. Missing sources and incomplete acquisition remain unknown, never measured zero demand.

## The Architectural Reality

Existing owners:

- `ai/services/memory-core/communityBatchContract.mjs`: neutral shape, hosted envelope, canonical digest.
- `CommunityBatchAdmissionService.mjs`: tenant/epoch fences, retained receipt replay, CAS and persisted coverage JSON.
- `CommunityActivityService.mjs`: snapshot-qualified coverage and bounded attention measurements.
- `ai/services/github-workflow/community/`: provider acquisition, normalization and assembly.
- `communityStewardWakeLease.mjs` and `ai/scripts/maintenance/observeCommunitySteward.mjs`: shared evaluation and finite zero-dispatch observation.
- `ai/mcp/server/memory-core/openapi.yaml`: strict hosted/public contracts.

The existing receipt stores coverage JSON but not the transient adapter/provider-state versions. The digest hashes the entire coverage object and a constant outer batch version. Validation precedes retained-receipt lookup. These facts make an incidental outer-version bump or new mandatory pre-replay validator unsafe.

Structure-map `npm run ai:structure-map -- --files --loc` completed successfully. A pure `communityCoverageContract.mjs` belongs beside `communityBatchContract.mjs`: same stateless neutral contract role, no service lifecycle or new directory.

Prescription checked: `communityBatchContract.mjs` owns shape/digest semantics; admission owns replay and epoch fencing; `readCoverage` owns consumer qualification. A caller flag, observer-side waiver, config leaf or provider poll in Memory Core would bypass those owners. The independent D20161 STEP_BACK and non-author approval establish this owner split.

## The Fix

Implement one coherent qualification feature:

1. Define one versioned coverage contract with a recognized exact limitation set bound to batch contract, resource family, adapter and provider-state versions. Persist that binding inside coverage. Separate acquisition gaps from required `providerLimitations`; reject missing, duplicate, extra, wrong-family and unknown declarations.
2. Publish one neutral gap-item definition covering the actual producer keys and types, including nullable cursor and numeric census details; derive or drift-check the OpenAPI component against it. Keep existing range-gap compatibility. No runtime external-reference framework or per-call schema-file I/O.
3. Preserve the outer v1 digest contract and retained legacy bytes. Introduce explicit qualification semantics within coverage; known identical legacy replay must still reach its original receipt. Do not synthesize bindings, reorder old arrays or promote historical receipts.
4. Update all three provider runners and lossless assembly. Fixed limitations become declarations only under the recognized new contract; every real traversal/access/verification failure stays an acquisition gap. Issue loss/attribution caveats are explicit too.
5. Qualify current-epoch three-family evidence at the selected admitted snapshot. Expose raw completeness separately from qualified interpretation where needed, keeping original legacy receipt meaning readable. No expected-family subset switch.
6. Carry bounded historical/actor measurements and qualification through public schemas and the shared evaluator/observer. Add no provider calls, claims or dispatches.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Batch coverage, digest and hosted gap item | ADR 0036 §2.1/§2.4; existing neutral contract | Persist exact version/family declaration binding; shared typed gap definition; new evidence participates in digest | Legacy JSON/digest/replay retained; unknown/new malformed declarations cannot qualify | Owning JSDoc + OpenAPI | Exact legacy digest/replay, changed-coverage conflict, schema parity and all producer gap shapes |
| Provider runner → assembly → admission | Existing GitHub community adapters; ADR 0036 §2.4 | Separate fixed limitations from actual acquisition gaps; preserve evidence losslessly | Caps, access failures, failed verification and unknown generation stay incomplete/unqualified | Adapter contract JSDoc | Exhaustive/capped/mutated/error producer controls through real admission |
| Coverage/count/measurement responses | `CommunityActivityService.readCoverage/readAttentionCounts`; ADR 0036 §2.6 | Three-family current-epoch recognized evidence; visible limitations, eligible-admitted upper bound, bounded historical slice and actual actor exclusions | Legacy/mixed/missing/degraded stays unqualified; response-bearing unknown-actor subset stays unknown absent bound policy | Strict public schemas + service JSDoc | Snapshot, epoch, mixed-generation, actor and late-reply controls |
| Shared empty evaluation and retained observer | Existing `evaluateStewardWake` and observer; ADR 0036 §2.6 | Qualified empty reports observed vacancy and false shadow would-wake | Missing/degraded does not acquire fabricated vacancy/wake/bytes | Evaluator/observer JSDoc | Same evaluator for empty/nonzero controls; no dispatch capability |

## Decision Record impact

**aligned-with ADR 0036**, as amended by #1328. No new ADR, tenant authority, threshold or configuration is introduced.

## Decision Record

**Required: ADR 0036 amendment, merged first** at `d02bdd77`. Source Discussion is resolved/graduated to the owning ADR leaf #1327. This runtime leaf implements that accepted record; it does not reopen or replace the design.

## Discussion Criteria Mapping

- B1 exact declaration binding → AC-1, AC-2, AC-3.
- STEP_BACK persisted binding and snapshot interpretation → AC-3, AC-4.
- STEP_BACK assembly/public strict schemas → AC-2, AC-7.
- Bounded D and unchanged actor/closer predicates → AC-5.
- STEP_BACK qualified-empty vacancy → AC-6.
- Actual producer/consumer proof and authority ceiling → AC-7, AC-8.

## Acceptance Criteria

- [ ] AC-1: Recognized version/family bindings require the exact deterministic provider-limitation set. Missing/duplicate/extra/wrong-family/unknown tuples fail closed; runtime acquisition failures cannot be relabelled.
- [ ] AC-2: Real Issue/PR/Discussion gap shapes pass neutral and compiled hosted validation, including nullable cursor, entity IDs and numeric census fields. Closed-key, authority, secret and prose refusals remain; schema drift is mechanically detected. Assembly preserves declarations/binding.
- [ ] AC-3: Existing legacy receipt/checkpoint JSON and known digests remain byte/meaning compatible; identical known replay succeeds after checkpoint advancement, changed coverage conflicts, and new qualification requires fresh post-contract admission. No historical binding retrofit.
- [ ] AC-4: Counts require all three expected families, ACTIVE/current epoch, recognized compatible generations, zero acquisition gaps and snapshot-bound evidence. Legacy complete-looking, mixed/unknown, missing or capped evidence stays unqualified.
- [ ] AC-5: Internal/rostered close resolution and actor eligibility stay unchanged. Late post-close replies remain eligible. Historical slice is labelled “latest admitted closure; reopen not observable”; unknown/unreviewed-actor exclusions use admitted reasons, with no invented response-bearing denominator or historical reclassification.
- [ ] AC-6: Qualified empty demand traverses the shared evaluator, reports actual observed vacancy and `shadowWouldWake:false`. Missing/degraded coverage retains unknown vacancy/wake/payload measurements.
- [ ] AC-7: Actual producer seams → assembly → real SQLite admission → strict public count → shared evaluation/observer controls prove the new path, not synthetic complete fixtures alone. Relevant schemas and JSDoc agree.
- [ ] AC-8: Live activation remains false, unmeasured thresholds unset; no source administration, acquisition run, credential transfer, deployment/restart, Task claim or wake dispatch occurs. ADR §2.1 receives the non-blocking batch-contract-version wording clarification identified in review 5676025638.

## Out of Scope

Actual operator-approved deployment/source bootstrap, Neo provider acquisition, retained live observation and independent calibration remain **Residual-Owner: #106**. This leaf's evidence target is L2; fixtures and CI do not discharge that window.

No unknown-closer waiver, blanket closed-container exclusion, synthetic Discussion reopen, provider polling in the neutral observer, new admin authority or family-selector shortcut.

## Avoided Traps

- Raising only an allowlist permits omission; require exact set binding.
- Bumping the existing outer version changes the constant-hashed legacy digest; version coverage explicitly.
- Mandatory new pre-replay validation strands known receipts; preserve the retained replay path.
- A schema repair alone is not qualification, and qualification alone is not an observation window.
- A raw unknown-actor reason is not proof the original policy considered that kind response-bearing.

## Related

Parent #106; ADR leaf #1327 / #1328; observer #1321 / #1322; [D20161](https://github.com/orgs/neomjs/discussions/20161).

Origin Session ID: 607243d1-f786-4b5c-b0a2-028defcc0310
Retrieval Hint: `D20161 exact declaration binding legacy receipt digest replay provider history gaps qualified empty`

Creation evidence: current source `3bc4a9a9`; parent106 native children all closed; open own-assignment bodies106/107 read (107 is separate zero-authority composition). Live all-state coverage search found no equivalent open runtime leaf. Final live latest-open sweep: latest20 open issues read at 2026-10-28 00:08:29 UTC, newest1325; no equivalent runtime leaf. All-state A2A30 read immediately before create; no competing scope claim. MC problem-noun sweep returned the existing106/D20161 custody memories38aa7ade and97723313; prior accepted decision is reused, not re-derived. KB ticket synthesis returned unrelated historical references, so it supplies no absence claim. Six open own assignments checked; same-surface106 and107 bodies read. Labels verified live. Epic sweep N/A: this is one non-epic implementation leaf.


## Timeline

- 2026-10-28T00:08:33Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-28T00:08:34Z @neo-gpt-euclid added the `enhancement` label
- 2026-10-28T00:08:34Z @neo-gpt-euclid added the `ai` label
- 2026-10-28T00:08:34Z @neo-gpt-euclid added the `testing` label
- 2026-10-28T00:08:35Z @neo-gpt-euclid added the `architecture` label
- 2026-10-28T00:08:47Z @neo-gpt-euclid added parent issue #106
- 2026-10-28T00:14:36Z @neo-gpt-euclid cross-referenced by #106
- 2026-10-28T01:13:31Z @neo-gpt-euclid cross-referenced by PR #1330

