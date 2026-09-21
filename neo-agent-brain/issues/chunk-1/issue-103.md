---
id: 103
title: Expose a temporal community Bird View and seen state
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
assignees:
  - neo-gpt
createdAt: '2026-07-14T05:31:18Z'
updatedAt: '2026-09-21T10:59:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/103'
author: neo-gpt
commentsCount: 1
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 105 Reconcile GitHub Discussions and nested replies'
  - '[x] 15153 Reconcile GitHub pull requests and reviews exhaustively'
  - '[x] 15152 Reconcile GitHub issue activity exhaustively'
  - '[x] 15151 Admit idempotent community batches into durable history'
blocking:
  - '[ ] 98 Prove the community-activity authority chain end to end'
  - '[ ] 3 Calibrate community policy from measured evidence'
closedAt: '2026-09-21T10:59:51Z'
---
# Expose a temporal community Bird View and seen state

## Context

OQ6 selects a provider-neutral temporal Bird View inheriting the shipped citation, coverage, pagination, and honest-degradation envelope. OQ5 keeps per-viewer seen state explicitly non-authoritative. These belong in one presentation/query leaf; Task claim remains separate.

This is one fully closeable PR leaf under Epic neomjs/neo-agent-brain#106. The live parent-child and blocked-by graph is authoritative; this body owns only this leaf's contract.

## The Problem

A live-query-only explorer is not durable completeness, while a cached narrative or global seen flag can become false authority. Automatic title/body delivery also introduces cross-tenant leakage and prompt-injection exposure.

## The Architectural Reality

The provider-neutral query/application surface belongs in Memory Core and injects the GitHub source adapter, following `PullRequestHistoryService.mjs` plus `temporalBirdViewEnvelope.mjs`. Prose drill-down reuses source-relative trust from the GitHub reconciliation foundation; the default temporal view is metadata/citations. The operation name cannot encode GitHub.

The Agent OS structure map was run on 2026-07-14. New service/script/test placement must use the named sibling-file-lift fast paths; no service logic moves into MCP server entrypoint directories.

## The Fix

Expose a paginated half-open-window community-activity Bird View with manifest/revision identity, coverage/gap fields, citation-backed drill-down handles, explicit notAuthority, and per-viewer idempotent seen markers. Explicit prose drill-down rechecks tenant/source-relative trust and hostile-content projection at read time.

Intake decisions against the landed ledger:

- The requested window filters occurrence time; a fixed admitted-sequence cutoff freezes pagination membership. The cursor includes the unique observation-row tie-breaker because a batch shares one admitted sequence. `sourceEventId` is the stable occurrence identity, not a provider delivery ID.
- The caller supplies an explicit positive record limit; this leaf invents no pagination default. Default results use stored attention eligibility and the first adapter's supported event vocabulary, excluding popularity even if a malformed source classified it eligible.
- Seen state keys tenant, authenticated viewer and occurrence. Viewer identity comes from request context, never a caller field or session ID. Seen changes neither result membership nor source-manifest revision by default; it is presentation metadata only.
- Explicit drill-down returns current provider content, with its current revision/read timestamp and citation, rather than pretending to retain historical prose. Null/ambiguous missing content is unknown; deletion requires explicit evidence.
- GitHub inline review comments use their numeric REST IDs; other supported content entities use verified GraphQL node identities. The returned repository must match the current registered source, and source lifecycle/epoch is rechecked after the fetch.
- Fresh source association/collaboration determines prose trust for this path. The global Neo roster is not a substitute for that source-relative evidence; the existing sanitizer remains the projection primitive.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Temporal Bird View | OQ6 + PR neomjs/neo#15131 | Provider-neutral query with citations, coverage, cursor, revision, degradation, notAuthority | Incomplete sources remain visible; synthesis cannot infer through gaps | OpenAPI/tool docs | Envelope, window, pagination, citation tests |
| Prose drill-down | OQ1/OQ8 STEP_BACK | Explicit source fetch after tenant/source trust projection | Denied/unavailable content returns metadata-only status | Trust docs | Cross-tenant, hostile-content, deletion tests |
| Seen state | OQ5 | Idempotent per-viewer presentation marker | Never affects claim, Task, count authority, or LifecycleFrontier | Service JSDoc | Viewer-isolation and authority-negative tests |

## Decision Record impact

Depends on ADR 0036, admission, and all GitHub reconciliation leaves; composes ADR 0035 without creating ranking or lifecycle authority.

## Decision Record

**Required: ADR 0036.** This leaf is not code-ready until the ADR-0036 child of neomjs/neo-agent-brain#106 is accepted at the human merge gate.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ5 | Seen is per viewer and non-authoritative; claim/resolve remain separate. |
| OQ6 | Inherits temporal Bird-View envelope, citation, coverage, pagination, and honest degradation. |
| OQ7 | Supplies the explicit query surface future Fleet readers consume without new authority. |
| OQ8 STEP_BACK | Prose requires tenant/source-relative trust and remains outside automatic paths. |

Source authority: Discussion #15139 body at the version-bound graduation anchor plus Grace's [STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120) and [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315).

## Operator Scope Clarification — 2026-07-14

The community substrate is **not** a mirror of every GitHub repository notification. It separates:

1. **source occurrences** needed to reconstruct supported issue, pull-request/review, and Discussion conversation state; and
2. **attention-eligible community items**: externally authored, response-bearing occurrences that may need maintainer attention.

Stars/un-stars, forks, watches, and equivalent popularity telemetry are outside the community-event source families and cannot enter Bird View, counts, wake, or Task claim. Internal/rostered actions may update or resolve the state of an existing external item without minting new community attention. First-time versus trusted-repeat external status affects trust/projection, not basic eligibility. Bot eligibility must be an explicit ADR disposition and cannot be inferred from provider actor kind or trust tier.

Attention eligibility remains zero-authority: it does not assign work, enter LifecycleFrontier, or create a Task. Only the explicit canonical claim transition owns that promotion.

## Acceptance Criteria

- [ ] **AC1** — The operation name and response schema are provider-neutral while GitHub is the first adapter.
- [ ] **AC2** — Half-open windows, stable pagination, source-manifest identity, coverage, gaps, and degraded reasons are explicit.
- [ ] **AC3** — Every synthesized/drill-down fact is citation-backed and `notAuthority` remains true.
- [ ] **AC4** — Default results contain metadata and handles, not automatically delivered title/body/excerpt prose.
- [ ] **AC5** — Explicit prose drill-down revalidates tenant, source, collaborator/trust, and hostile-content projection.
- [ ] **AC6** — Per-viewer seen writes are idempotent and isolated across viewers/tenants.
- [ ] **AC7** — Seen cannot change unclaimed counts, create/bind Tasks, enter LifecycleFrontier, or rank Golden Path.
- [ ] **AC8** — Deleted/inaccessible/unknown content remains distinguishable without retaining stale prose.
- [ ] **AC9** — MCP/SDK schemas and focused tests cover pagination, coverage, denial, and independent degradation.

- [ ] **AC10** — Default community results contain only attention-eligible external items; internal/rostered occurrences may explain current resolution state but never appear as new attention rows.
- [ ] **AC11** — Star/un-star, fork, watch, and equivalent popularity events are negative fixtures and never appear in results, counts, seen state, or drill-down handles.

## Out of Scope

Task claim, count/wake projection, Fleet UI, cached Bird-View narratives, ranking, or automatic assignment.

## Avoided Traps

Do not name the operation after GitHub, reuse live provider query as completeness, globalize seen, synthesize through gaps, or place prose in hooks.

## Related

- Parent: neomjs/neo-agent-brain#106
- Source: Discussion #15139 OQ5-OQ8
- Precedents: neomjs/neo#15088, PR neomjs/neo#15131, neomjs/neo#13046, PR neomjs/neo#13048

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a

## Handoff Retrieval Hints

- `provider neutral community Bird View seen non authority`
- `tenant source relative prose drilldown citations coverage`


## Creation Freshness

Creation duplicate sweep: immediately before filing at 2026-07-14T05:31:17.761Z, checked the latest 20 open issues and last 30 all-state A2A messages. The independent broader audit at 2026-07-14T05:13:00Z covered open and closed issues, pull requests, A2A, ADRs, and code; no equivalent owner or foreign claim existed.


## Timeline

- 2026-07-14T05:31:19Z @neo-gpt added the `enhancement` label
- 2026-07-14T05:31:20Z @neo-gpt added the `ai` label
- 2026-07-14T05:31:20Z @neo-gpt added the `architecture` label
- 2026-07-14T05:31:20Z @neo-gpt added the `security` label
- 2026-07-14T05:32:50Z @neo-gpt marked this issue as being blocked by #15151
- 2026-07-14T05:32:51Z @neo-gpt marked this issue as being blocked by #15152
- 2026-07-14T05:32:53Z @neo-gpt marked this issue as being blocked by #15153
- 2026-07-14T05:34:32Z @neo-gpt cross-referenced by #106
- 2026-07-22T11:43:28Z @neo-gpt-emmy cross-referenced by #15524
- 2026-08-24T11:39:12Z @neo-gpt marked this issue as blocking #3
- 2026-08-26T15:13:33Z @neo-gpt marked this issue as being blocked by #15152
- 2026-08-26T15:13:33Z @neo-gpt marked this issue as being blocked by #15151
- 2026-08-26T15:13:34Z @neo-gpt marked this issue as being blocked by #15153
- 2026-08-26T15:13:34Z @neo-gpt marked this issue as blocking #3
- 2026-09-19T23:01:07Z @neo-gpt cross-referenced by PR #392
### @neo-gpt - 2026-09-20T00:54:08Z

Intake has started against the landed admission/reconciliation substrate. Two source constraints will shape the read implementation:

- `mc_community_observation.admitted_sequence` identifies a batch position, not one unique observation. Multiple rows may share both that sequence and `occurred_at`. A SQLite control with two such rows gives zero remaining rows under a sequence-only continuation after the first result, versus one under `(admitted_sequence, observation_row_id)`. Stable pagination therefore needs a unique tie-breaker and a fixed admitted-sequence snapshot; a sequence alone is not a cursor.
- `listObservations()` is tenant-scoped but unpaginated. The new query belongs beside the operational ledger with an in-query tenant predicate, rather than loading all rows and paginating a cached narrative. The existing `buildTemporalBirdViewEnvelope()` already supplies manifest/revision citations, coverage/degradation, and `notAuthority: true`; reuse that contract.

The default projection will use the stored attention disposition and metadata only. Seen writes need their own tenant + authenticated-viewer + occurrence key and must not update observations, receipt/checkpoint state, or future Task bindings. Explicit prose requires a fresh source-relative trust read; the stored association is historical evidence, not permanent permission.

No implementation branch or claim yet. The half-open window's event-time semantics and server-derived viewer identity are being checked against the existing temporal and request-context contracts before the public schema is fixed.

- 2026-09-20T01:37:04Z @neo-gpt assigned to @neo-gpt
- 2026-09-20T02:05:34Z @neo-gpt cross-referenced by PR #396
- 2026-09-21T10:43:27Z @tobiu referenced in commit `86da615` - "fix(memory-core): retain community content disclosure (#103)"
- 2026-09-21T10:59:51Z @tobiu referenced in commit `bb52842` - "Merge pull request #396 from neomjs/codex/103-community-bird-view

feat(memory-core): expose community activity and seen state (#103)"
- 2026-09-21T10:59:52Z @tobiu closed this issue

