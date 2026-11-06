---
id: 1501
title: Implement replacement-proven legacy core-row retirement
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-11-06T03:36:41Z'
updatedAt: '2026-11-06T14:03:19Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1501'
author: neo-gpt-euclid
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
blocking:
  - '[ ] 1507 Use the core-row retirement operation, then remove its one-shot code'
  - '[ ] 419 Retire replaced legacy core rows by profile receipt'
closedAt: '2026-11-06T14:03:19Z'
---
# Implement replacement-proven legacy core-row retirement

## Context

This is the single source-operation leaf of #419, which retains the deployed migration and its receipts. The existing issue joins two independently deliverable obligations: implement a safe maintenance tool, then activate/materialize the replacement profiles and use that tool on the selected plane. The canonical profile manifest was absent in the read-only November 6 intake recorded on #419; source tests cannot certify the deployed migration.

The [accepted equality boundary](https://github.com/neomjs/neo-agent-brain/issues/419#issuecomment-6675295661) proves redundancy against landed replacements without requiring historical repository-origin reconstruction. Vega's independent handoff read recommended this source/deployment split. This is one coherent operation, not separate planner/executor/CLI tickets.

## The Problem

The additive core-profile writer uses new extraction-bound IDs, so unchanged facts can coexist with legacy IDs. A tuple-wide stale delete would also remove protected families and other repositories. The conversation-only operation from #417 explicitly cannot retire code rows.

## The Architectural Reality

At Brain `87ad3580a7bcb99a7109bb9cd10bf1c0850eedfc`, `coreCorpusProfileRunner.mjs` owns manifest bytes and current-revision validation; `VectorService` owns expansion and tenant-aware IDs; `chunkRowMetadata.mjs` owns stored metadata. The manual maintenance sibling is `ai/scripts/maintenance/retireLegacyConversationRows.mjs`. The shared heavy-maintenance lease already serializes this class of operation; `atomicFileWrite.mjs` supplies strict file-and-directory fsync.

## The Fix

Deliver one manual `retireLegacyCoreRows.mjs` entrypoint, a pure correspondence/planning module, an observed-input adapter and a resumable executor in `ai/scripts/maintenance/`, with mirrored Playwright unit specs. A dry run captures a private plan and exact IDs. Apply reloads and revalidates that plan, current source artifacts and actual target rows, and deletes only the captured proven old IDs.

Protected families are excluded before matching. A match requires exact normalized path/name/kind and equality of every non-provenance field, including unknown meaning fields. The finite path/type conversion table is bound into correspondence bytes. Identical shared-path facts may map to both repositories; every selected split child must be present and reassemble the same fact. Changed or unmatched facts stay.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Profile and row observations | Existing manifest/revision readers, VectorService expansion/stamping, metadata writer | Bind exact source bytes/revisions and actually landed IDs; observe foreign population as IDs only | Absent/stale/changed inputs refuse | Module JSDoc | AC-1/2 |
| Manual CLI | This leaf; #419 operational boundary | `--plan-file` required; dry run default; `--apply --confirm <planDigest>` required for deletion; optional exact checkout revision and batch size | No bootstrap, embedding or scheduler activation | CLI help and module usage | AC-1/3 |
| Private plan and journal | Existing atomic writer with strict fsync | Private regular files; exact digest; pending persisted before delete; read-reconciled retry | Preserve uncertain progress and return bounded failure receipt; no automatic restore | Module JSDoc | AC-3/4 |
| Preservation receipt | Captured whole population and old metadata | Exact protected IDs/count/digest remain; candidate mutation cannot widen delete set | Refuse before dispatch when possible; explicit incomplete receipt after dispatch | Module JSDoc | AC-4 |

## Decision Record impact

Aligned with ADR 0014 §5.2: shared core acquisition remains separate from tenant ingestion. No new daemon, transport, config leaf or activation authority.

## Acceptance Criteria

- [ ] **AC-1 — bounded preview:** the operation reads the selected validated two-profile manifest and current reader revisions, observes actual derived target IDs, and records a private exact-ID plan plus bounded family counts, profile bindings and preservation digest. Zero legacy rows produce an explicit no-op. No embedding, collection bootstrap or scheduler runs.
- [ ] **AC-2 — redundancy proof:** all non-provenance equality and declared identity conversions govern certification. Tests cover changed unknown fields, same-path Engine/Brain alternatives, generic one-line cross-file refusal, complete/missing/tampered split children and protected conversation/Skill/release/concept/foreign/current-profile rows.
- [ ] **AC-3 — explicit resumable execution:** apply requires exact plan confirmation and the existing maintenance lease; it rechecks source/map bytes, old metadata and landed targets. Pending-before-delete and durable progress support partial failure, lost delete/journal acknowledgement, completed retry and confirmed-ID resurrection refusal. Borrowed plans or journal callbacks cannot extend deletion IDs.
- [ ] **AC-4 — isolated native preservation:** the canonical Brain unit configuration runs a real temporary Chroma control that deletes only captured old IDs, preserves replacement/foreign metadata and exact IDs, and retries without deleting again. Deployed plane execution and runtime-import initialization are not inferred from this fixture.

## Out of Scope

#419 keeps the deployed baseline, profile materialization/embedding decision, restorable-backup gate, actual migration run, unmatched-family disposition, post-run KB query receipts and activation handoff. No source merge closes #419. No shared-plane deletion, provider activation, scheduler enablement, profile/credential change or service restart is authorized by this leaf. Production service imports may initialize recorder telemetry after lease acquisition; isolated executor tests do not certify that import stack.

## Avoided Traps

No broad stale pass or shadow swap. No origin-reconstruction requirement after a full redundancy witness. No path/name-only proof, incomplete child population or approval flag as deletion authority. No private plan contents in public artifacts. The source/deployment split preserves the existing open operational owner instead of closing it with fixture evidence.

## Related

#419 (deployed migration owner), #282 (additive profile writer), #417 (conversation-only predecessor).

Ownership: @neo-gpt-euclid, continuing the already claimed #419 operation; excluded from contributor pickup because implementation is already owned.

Origin Session ID: 5be6d6e3-45c3-458f-abb2-b7d09587111d
Retrieval Hint: `legacy core rows replacement equality exact-ID retirement419 source deployed custody`

Structure map: Brain `ai/scripts/maintenance/` sibling `retireLegacyConversationRows.mjs`; `ai:structure-map -- --files --loc` executed.

Live latest-open + closed sweep: latest20 open Brain issues (created-descending) + closed search `legacy core` at2026-11-06 03:36UTC; #417's code-row exclusion and #282's no-execution boundary read. #419 remains the existing deployed owner; no equivalent source-only execution leaf found. A2A in-flight sweep: latest30 across all read states through03:36UTC; only Euclid's existing419claim and Vega's split recommendation overlap. MC sweep: duplicate code rows / changed profileIDs returned off-topic history; KB's ranked primary #282 points to419. Own-assignment sweep: six open Brain assignments;419 and84 bodies read,84 is parity cleanup and supplies no code-row retirement operation.


## Timeline

- 2026-11-06T03:36:41Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-11-06T03:36:42Z @neo-gpt-euclid added the `enhancement` label
- 2026-11-06T03:36:42Z @neo-gpt-euclid added the `ai` label
- 2026-11-06T03:36:42Z @neo-gpt-euclid added the `architecture` label
- 2026-11-06T03:36:43Z @neo-gpt-euclid added the `agent-os` label
- 2026-11-06T03:37:10Z @neo-gpt-euclid marked this issue as blocking #419
- 2026-11-06T03:37:42Z @neo-gpt-euclid cross-referenced by #419
- 2026-11-06T03:48:39Z @neo-gpt-euclid cross-referenced by PR #1502
- 2026-11-06T03:56:53Z @neo-opus-vega cross-referenced by #1435
- 2026-11-06T04:00:22Z @neo-gpt-euclid referenced in commit `57a0ec1` - "fix(kb): recheck replacement proof at completion (#1501)"
- 2026-11-06T04:34:18Z @neo-gpt-euclid referenced in commit `ee775ff` - "fix(kb): honor the selected core corpus root (#1501)"
- 2026-11-06T12:57:41Z @neo-gpt-euclid cross-referenced by #1507
- 2026-11-06T12:58:18Z @neo-gpt-euclid marked this issue as blocking #1507
- 2026-11-06T14:03:18Z @tobiu referenced in commit `281b8ee` - "feat(kb): retire exact replacement-proven legacy core rows (#1501) (#1502)

* feat(kb): retire exact replacement-proven legacy core rows (#1501)

* fix(kb): recheck replacement proof at completion (#1501)

* fix(kb): honor the selected core corpus root (#1501)"
- 2026-11-06T14:03:20Z @tobiu closed this issue

