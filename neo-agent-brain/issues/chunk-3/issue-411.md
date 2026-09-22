---
id: 411
title: Activate the github-content-sync KB tenant on the deployed plane
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-09-22T22:25:56Z'
updatedAt: '2026-09-22T22:25:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/411'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 237 A ref-not-found is retried as a transient, 36 times and counting'
blocking: []
---
# Activate the github-content-sync KB tenant on the deployed plane

## Context

#402 delivers the code half of the operator's top item (2026-09-21: `ask_knowledge_base` and `query_documents` unusable, stale for a month): the `ConversationCorpusSource` extractor and the tenant declaration for `neomjs/github-content-sync` in `deploy/cloud/kb-config.yaml`. That declaration ships `disabled: true`. This ticket is the activation half — the plane-side acceptance criteria no PR can satisfy from inside a sandbox — split out of #402 so the code leaf can close on its code.

## The Problem

The tenant poller (`tenant-repo-sync`) is not healthy. #237's specimen `453ffb0965b3` has retried a ref-not-found as a transient 215+ times (2026-09-19 read) with `accessReadiness: degraded`, and at 2026-09-22T21:59Z the lane holds the lease and defers `core-corpus-projection`, `dream` and `graphlog-compaction` (@neo-gpt-emmy's read-only healthcheck). A tenant enabled into that state inherits it, and its failure would read as the extractor's. The deployed plane also still runs Engine revision `467fd122` (2026-08-25, pre-split), so nothing merged into the Brain reaches it until #253's cut.

## The Architectural Reality

- `deploy/cloud/kb-config.yaml` — tier 2 of tenant-config resolution, mounted read-only into the orchestrator and kb-server (`deploy/cloud/docker-compose.local-agent-os.yml:48`, `:89`; documented for the cloud compose at `docker-compose.yml:529-536`). The `github-content-sync` entry carries the extraction profile and `disabled: true`.
- `ai/daemons/orchestrator/services/TenantRepoSyncService.mjs:312` — `repo.disabled === true` skips the entry. Removing the flag is the whole activation.
- `NEO_ORCHESTRATOR_TENANT_REPO_SYNC_ENABLED` — the lane's master toggle: cloud profile default-on when `tenantRepos[]` is configured, local profile default-off (`learn/agentos/DeploymentCookbook.md`). The operator's local dockerized Agent OS needs the toggle as well as the flip.
- Freshness surfaces: `IngestionService.getTenantManifest({tenantId, repoSlug})` (the corpus-owned manifest), the extraction receipt bound to a `github-content-sync` revision, and `ask_knowledge_base` answers that cite conversations with their origin.

## The Fix

1. Disposition #237's specimen — closed, or a waiver by its owner recorded here.
2. One-line PR: remove `disabled: true` from the `github-content-sync` entry; for the local profile, turn the tenant-sync toggle on in the local compose or document the operator step.
3. Deploy (or the operator's local rebuild), wait one sweep, record the receipts here.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `deploy/cloud/kb-config.yaml`, `github-content-sync` entry | tenant-repo access contract (`normalizeTenantRepoEntry` spreads the entry; `disabled` read at `TenantRepoSyncService.mjs:312`) | flag removed → the poller mirrors and ingests the corpus on its sweep | flag present → no ingestion; never a fallback to the Engine tree | `learn/agentos/cloud-deployment/TenantIngestionModel.md` | AC-3 manifest + receipt |

## Decision Record impact

`aligned-with` D#17846 §8.6 / §8.6a / §8.7 and ADR 0019; nothing amended. Depends on #237 (native `blocked_by`).

## Acceptance Criteria

- [ ] **AC-1** — #237's `453ffb0965b3` specimen is dispositioned before the flip: the ticket is closed, or its owner records a waiver here naming what the new tenant inherits.
- [ ] **AC-2** — `disabled: true` is removed from the `github-content-sync` entry, and the local Agent OS profile has the tenant-sync toggle on or a documented operator step. The PR carries `Refs` until AC-3 and AC-4 are recorded here, then `Resolves`.
- [ ] **AC-3** — *(deployed plane)* a corpus-owned manifest (`tenantId: 'neo-shared'`, `repoSlug: 'github-content-sync'`) exists and the extraction receipt is bound to a `github-content-sync` revision — recorded here before any freshness claim.
- [ ] **AC-4** — *(deployed plane, after AC-3)* `ask_knowledge_base` cites a `neomjs/neo` conversation created the same day and a `neo-agent-brain` conversation, each showing its origin.

## Out of Scope

- The extractor, the declaration and the retirement of the three Engine-tree Sources — #402.
- The poller's starvation and the blobless-mirror cost — #64, #65.
- The image cut — #253.

## Avoided Traps

- ⛔ Do not enable the tenant "to see whether it works" while #237's lane is in its failure state: the new entry inherits the back-off and the lease contention.
- ⛔ Do not claim freshness from a green sweep. The manifest and the revision-bound receipt are the evidence; `ask` citing today's conversation is the outcome.

## Related

#402 (code leaf) · #237 (readiness gate) · #253 · #64 · #65 · #246 / PR #410 (the FM feed over the same corpus) · neomjs/neo#17416 (epic) · D#17846 §8.6, §8.7

Live latest-open sweep: latest 20 open Brain issues at 2026-09-22T22:15Z — none equivalent (nearest #237, #64, #253). A2A in-flight sweep (30 most recent, all read-states, 22:26Z): no claim on tenant activation; #402 ownership confirmed by @neo-gpt-emmy (21:57Z) and @neo-fable-clio (22:01Z). Memory Core sweep: no prior decision beyond D#17846 §8.7 step 1. Own-assignment sweep: #402, #362, #237, #23, #64, #65 — none equivalent. Structure map: N/A — no `.mjs` placement; one config line under `deploy/cloud/`.

Origin Session ID: fc04c361-0cae-4a80-9506-fa2ef4785d2b
Retrieval Hint: "github-content-sync tenant activation disabled flag kb-config deployed plane freshness receipt #237 specimen"

## Timeline

- 2026-09-22T22:25:57Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-22T22:25:58Z @neo-opus-vega added the `enhancement` label
- 2026-09-22T22:25:58Z @neo-opus-vega added the `ai` label
- 2026-09-22T22:25:59Z @neo-opus-vega added the `agent-os` label
- 2026-09-22T22:26:31Z @neo-opus-vega marked this issue as being blocked by #237
- 2026-09-22T22:28:31Z @neo-opus-vega cross-referenced by #402
- 2026-09-22T22:30:07Z @neo-opus-vega cross-referenced by PR #412
- 2026-09-22T22:48:39Z @neo-fable cross-referenced by #19057
- 2026-09-22T22:50:21Z @neo-fable cross-referenced by #19058

