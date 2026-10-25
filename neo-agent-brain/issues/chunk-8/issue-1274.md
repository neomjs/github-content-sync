---
id: 1274
title: Raw era chains must reject reversed and empty closed intervals
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-25T17:00:05Z'
updatedAt: '2026-10-25T18:04:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1274'
author: neo-gpt-sophie
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
blocking:
  - '[ ] 700 An auto-provisioned agent identity carries no model family, so family-keyed budgets, aliases and wakes skip it'
closedAt: '2026-10-25T18:03:23Z'
---
# Raw era chains must reject reversed and empty closed intervals

## Context and problem
While qualifying #700's temporal reader boundary, an exact-source probe found that `validateEraChain` accepts a raw closed episode with `since=2026-10-02T00:00:00Z`, `until=2026-10-01T00:00:00Z`, followed by a valid open episode starting October 3. Equal endpoints also return `{valid:true, reason:null}`. The builder refuses both. This is a synthetic reproduction, not a claim that a live resident has corrupt history.

## Architectural reality
Design authority: `createEmbodiedEpisodeNode` requires a closed era's end to be after its start (`ai/graph/identitySchema.mjs:101–102`). The chain validator's own comment at 146–148 explicitly covers raw persisted nodes rather than trusting builder provenance. It currently checks finite timestamps but omits each closed interval's positive duration.

The earlier [raw-timestamp repair](https://github.com/neomjs/neo/pull/14729) established the same emit/accept boundary. Hydration and migration consume this validator; repairing only the builder or adding reader-specific guards leaves the shared accepting boundary unsound.

## Fix
Enforce `until > since` in the existing per-era validation loop, after timestamp parseability checks. Extend the existing `test/playwright/unit/ai/graph/identitySchema.spec.mjs` with raw-node regression controls. Keep adjacent eras and gaps valid. No new module or schema representation is needed.

| Surface | Authority | Behaviour / refusal | Evidence |
|---|---|---|---|
| Raw closed episode in `validateEraChain` | Existing builder's temporal contract | Refuse reversed and zero-duration intervals before inter-era ordering | Plain-object regressions |
| Valid chain and `migrateEra` | Existing non-overlap and pure-migration contract | Keep adjacent/gapped valid history; refuse migration from malformed history without mutation | Boundary and migration controls |
| Hydration/render consumers | Existing validator delegation | Continue to reject an invalid chain through their current path | Focused existing suites |

## Acceptance criteria
- [x] Reversed and zero-duration raw closed eras refuse, including equal instants represented by different ISO offsets.
- [x] Positive-duration adjacent eras and positive gaps remain accepted.
- [x] Migration from either malformed chain refuses without mutating the supplied history.
- [x] Focused schema, hydration and render-contract tests pass; the new raw-node regressions fail before the repair.

## Scope and rationale
This is a bounded correctness prerequisite for #700, independent of its ungraduated transport decision. No family policy, operator admission, model-null schema change, storage migration, reader reimplementation or new runtime/configuration surface is included. Do not call the builder wholesale from the validator: that broadens the change into unrelated field validation.

Decision Record impact: aligned with the existing identity-era contract; no authority amendment.

## Discovery
Exact reproduction at `1bce07ce`; source and existing spec unchanged at fetched `418e62d3`. `npm run ai:structure-map -- --files --loc` passed: existing owner `ai/graph/identitySchema.mjs`, sibling schema modules.
Live latest-created 20 open issues and latest 30 all-state A2A messages checked immediately before filing on 2026-10-25; no equivalent claim. All-state symbol/interval searches and open-PR search found no fix. MC's precise raw-validator query recovered the earlier emit/accept repair; the broader final query returned unrelated results. KB returned adjacent #114 with a context-budget caveat, not proof of absence. Own assignments: three open; #700's body explicitly excludes schema work, and the other two cover unrelated outcomes. #114 is the broader schema epic, not this defect.

Related: #700, #114. Defect capture: MC `a27dec97-dea8-47d7-bc35-2019c1b07aa5`.
Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Retrieval Hint: "validateEraChain raw closed era reversed zero duration until since"

Delivered by PR #1275, merged by @tobiu on 2026-10-25 at 18:03:22Z as `3c4f22735b8e257bcc843777bd5783b51fee58a8`. The two changed files match approved `03dab8a5` exactly. Grace review 5644320945 independently reproduced the red-before-fix controls; required CI and focused schema/hydration/render checks passed. This closes the validator leaf, not #700.


## Timeline

- 2026-10-25T17:00:05Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-25T17:00:06Z @neo-gpt-sophie added the `bug` label
- 2026-10-25T17:00:06Z @neo-gpt-sophie added the `ai` label
- 2026-10-25T17:00:07Z @neo-gpt-sophie added the `agent-os` label
### @neo-gpt-sophie - 2026-10-25T17:01:21Z

Prescription checked: `ai/graph/identitySchema.mjs` owns the concern. An independent bounded source read confirmed that hydration and migration already delegate here; placing guards in those consumers would duplicate the invariant. The existing per-era loop is the narrow fix, after finite timestamp validation, with no builder-wide revalidation or model-null change. Same-session authored ticket: creation reasoning retained; this records the non-author prescription check. The new regressions will run red before the source change.

- 2026-10-25T17:05:38Z @neo-gpt-sophie cross-referenced by PR #1275
- 2026-10-25T17:10:47Z @neo-gpt-sophie marked this issue as blocking #700
- 2026-10-25T18:03:23Z @tobiu referenced in commit `3c4f227` - "fix(identity): reject nonpositive closed eras (#1274) (#1275)"
- 2026-10-25T18:03:24Z @tobiu closed this issue

