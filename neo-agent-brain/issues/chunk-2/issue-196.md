---
id: 196
title: Delete extraction-only machinery and its tests
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
  - testing
  - agent-os
  - tech-debt
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-27T15:06:37Z'
updatedAt: '2026-08-28T16:45:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/196'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 191
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
closedAt: '2026-08-28T16:45:55Z'
---
# Delete extraction-only machinery and its tests

## Context

Brain #191 owns the deletion-first reset under parent Epic #189. The completed repository cut left temporary extraction instruments and their dedicated tests in the product tree.

Live revalidation against Brain `dev` at `7516d9b` identified an exact ten-path family totaling 10,888 lines. These files exist only to decide or prove the completed Engine-to-Brain cut; none has a package command, workflow, runtime import, current runbook, or other production caller outside the family itself.

## The Problem

One-shot extraction machinery now reads as supported product capability. Its tests preserve it indefinitely even though the cut did not use the machinery as final execution authority. Keeping it increases install size, maintenance cost, and false architectural authority.

## The Architectural Reality

The exact removable family is:

- `ai/scripts/diagnostics/agentOsExtractionInventory.mjs`
- `ai/scripts/diagnostics/agentOsExtractionInventory.json`
- `ai/scripts/diagnostics/agentOsPlaneBoundaryProof.mjs`
- `ai/scripts/diagnostics/devDependencyCensus.mjs`
- `ai/scripts/diagnostics/planePlacementCensus.mjs`
- `migration/learn-custody-census.v1.json`
- `test/playwright/unit/ai/scripts/diagnostics/agentOsExtractionInventory.spec.mjs`
- `test/playwright/unit/ai/scripts/diagnostics/devDependencyCensus.spec.mjs`
- `test/playwright/unit/ai/scripts/diagnostics/planePlacementCensus.spec.mjs`
- `test/playwright/unit/ai/services/agentOsPlaneBoundary.spec.mjs`

The caller-analysis positive controls are the surfaces that must stay:

- `scriptPlaneClosure.mjs` is imported by `lint-script-plane.mjs`, which runs in `script-plane-lint.yml`.
- `denyCloudPlanePackages.loader.mjs` remains the runtime-denial primitive. Its host/cloud denial controls in `hostBarrelRuntimeReach.spec.mjs` pass; the separate shared-fixture arm is already base-red because Brain lacks `test/playwright/fixtures.mjs`, and belongs to projection-removal ticket #198.
- `migrationCensusReport.mjs` is a recurring Cloud operator surface exposed by `deploy/cloud/package.json` and documented by `MultiTenantMigrationGuide.md`.
- `consumerRelevanceCensus.mjs` / `consumerRelevanceMap.mjs` are not coupled to the extraction family and are not dispositioned by this leaf.

The original ticket body incorrectly grouped `migrationCensusReport.mjs` with extraction-only artifacts. Live caller evidence falsified that classification; this body corrects it before implementation.

## The Fix

Delete the exact ten-path family in one PR. Remove the retained `hostBarrelImportReach.spec.mjs` comment that cites `planePlacementCensus.mjs` as a live evidence coordinate while preserving the parsing rationale it records. Do not replace any deletion with a retirement registry, archive manifest, successor diagnostic, or tombstone.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| exact ten-path extraction family | completed cut + live caller graph | absent from Brain product tree | git history remains provenance | remove live references | deleted paths + zero live refs |
| standing static-closure lint | `lint-script-plane.mjs` + workflow | retained unchanged | n/a | existing source | focused lint run |
| standing runtime-denial primitive | loader + host/cloud controls in `hostBarrelRuntimeReach.spec.mjs` | retained unchanged | shared-fixture arm stays #198-owned | existing source | focused unit run |
| recurring Cloud migration census | nested Cloud package + operator runbook | retained unchanged | n/a | existing runbook | caller/reference proof |

## Decision Record impact

Aligned with accepted ADR 0040. The extraction wrapper retires after the completed cut; ADR 0040's standing static-closure and runtime-denial lineages remain intact. No topology decision changes.

ADR successor-risk: `adr-aligned` — artifact #196 (2026-08-27); ADR 0040 accepted 2026-08-23; current source proves the standing witnesses remain; route `continue`.

## Acceptance Criteria

- [ ] The exact ten extraction-only scripts, committed outputs, and dedicated tests listed above are deleted.
- [ ] `scriptPlaneClosure.mjs`, `lint-script-plane.mjs`, and the Script Plane Lint workflow remain functional.
- [ ] `denyCloudPlanePackages.loader.mjs` and the retained host/cloud runtime-denial controls remain functional; the pre-existing shared-fixture failure stays unchanged and #198-owned.
- [ ] `migrationCensusReport.mjs`, its Cloud package command, and its operator runbook remain unchanged.
- [ ] No production source, package command, workflow, current guide, or retained test comment references a deleted artifact.
- [ ] The PR adds zero replacement audit, ledger, manifest, migration, diagnostic, or tombstone files.
- [ ] Relevant retained lint/unit checks pass.

## Intake Classification

- Ticket age: created 2026-08-27; revalidated 2026-08-28.
- Bot stale-band: not applicable — this repository has no close-inactive workflow; no `stale` or `no auto close` label is present.
- Currency / successor-risk: no open or merged PR closes #196; all ten paths remain on `dev`; current callers narrowed the deletion set before claim; #198 owns the retained shared-fixture projection failure.
- Parent review gate: #191 is self-authored, so its author cannot independently review it; goal Epic #189 has an independent structured review by Vega.
- Verdict: `valid-as-written` after this source-bound body correction.

## Out of Scope

Other migrations or diagnostics with a current repeated operator use; root Host-Edge command redesign; general source relocation; generic census cleanup beyond the exact family.

## Avoided Traps

- Keeping files because git history might be useful.
- Replacing deletions with tombstones or a deleted-file registry.
- Removing the standing plane-boundary primitives merely because the extraction wrapper composed them.
- Deleting a recurring operator tool because its filename contains `migration` or `census`.

## Related

Parent: #191 · Goal: #189 · Related: #198, neomjs/neo#17500

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: `delete exact AgentOS extraction inventory boundary proof plane census custody artifacts tests`


## Timeline

- 2026-08-27T15:06:39Z @neo-gpt-emmy added the `enhancement` label
- 2026-08-27T15:06:39Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:06:39Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:06:40Z @neo-gpt-emmy added the `testing` label
- 2026-08-27T15:06:40Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:06:40Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T15:40:56Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-28T15:48:49Z @neo-gpt-emmy cross-referenced by PR #207
- 2026-08-28T16:45:55Z @tobiu referenced in commit `0868c28` - "Merge pull request #207 from neomjs/codex/196-delete-extraction-machinery

chore(brain): delete extraction-only machinery (#196)"
- 2026-08-28T16:45:55Z @tobiu closed this issue
- 2026-08-31T00:18:29Z @neo-gpt-emmy cross-referenced by #191

