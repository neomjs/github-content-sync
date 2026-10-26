---
id: 101
title: Project bounded tenant community-attention counts
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - security
assignees:
  - neo-gemini-maryam
createdAt: '2026-07-14T05:32:00Z'
updatedAt: '2026-10-26T22:38:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/101'
author: neo-gpt-euclid
commentsCount: 1
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 2
  signals: []
blockedBy:
  - '[ ] 107 Live Lane Awareness — Wave-1 composition'
  - '[x] 102 Bind community events to canonical A2A Tasks'
  - '[x] 105 Reconcile GitHub Discussions and nested replies'
  - '[x] 15153 Reconcile GitHub pull requests and reviews exhaustively'
  - '[x] 15152 Reconcile GitHub issue activity exhaustively'
blocking:
  - '[ ] 98 Prove the community-activity authority chain end to end'
  - '[ ] 100 Enable metric-gated community-steward wake leases'
  - '[ ] 3 Calibrate community policy from measured evidence'
closedAt: '2026-10-26T22:38:31Z'
---
# Project bounded tenant community-attention counts

## Context

Staged option J requires a bounded tenant/source count-only projection before any steward wake. It is an attention signal, never ranking, assignment, or completeness authority.

This is one fully closeable PR leaf under Epic neomjs/neo-agent-brain#106. The live parent-child and blocked-by graph is authoritative; this body owns only this leaf's contract.

## The Problem

Even a count leaks another tenant's activity volume. Adding titles/excerpts creates hostile-content exposure, and rebuilding hook transport here would violate ADR 0035's projection-writer ownership.

## The Architectural Reality

Memory Core owns the count/query source. It may feed ADR 0035's typed projection writer only through that writer's admitted channel contract; it must not recreate `lifecycle-state.json`, a new file writer, or hook-time network access. Epic neomjs/neo-agent-brain#107 is the external projection-owner dependency until its writer leaf exists.

The Agent OS structure map was run on 2026-07-14. New service/script/test placement must use the named sibling-file-lift fast paths; no service logic moves into MCP server entrypoint directories.

## The Fix

Add a fixed, bounded count/oldest-age metadata projection over eligible unclaimed admitted events, scoped by server tenant/source, with explicit coverage and measurement fields. Exclude claimed/ineligible events using the neutral ledger and canonical Task binding. Expose no prose and no unmeasured escalation threshold.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Count query/channel | OQ7 J-stage1 | Bounded tenant/source counts plus fixed metadata and coverage | Missing/degraded sources stay explicit | Schema docs | Bounds/isolation tests |
| Eligibility | Admission + claim authority | Counts only eligible unclaimed items by explicit schema | Seen does not remove; claim does | JSDoc | State-matrix tests |
| Projection integration | ADR 0035 | Submit typed channel to existing writer owner | Writer unavailable means query still honest; no legacy fallback | ADR link | Static dependency and no-file-writer tests |

## Decision Record impact

Depends on ADR 0036, reconciliation, canonical Task binding, and ADR 0035 projection ownership.

## Decision Record

**Required: ADR 0036.** This leaf is not code-ready until the ADR-0036 child of neomjs/neo-agent-brain#106 is accepted at the human merge gate.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ7 | Ships J stage 1 count-only projection before leased wake. |
| OQ8 | Reapplies tenant/source isolation and excludes prose. |
| OQ5 | Uses canonical claim state; per-viewer seen remains non-authority. |
| OQ10 | Emits measurement inputs but introduces no age/wake threshold. |

Source authority: Discussion #15139 body at the version-bound graduation anchor plus Grace's [STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120) and [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315).

## Operator Scope Clarification — 2026-07-14

The community substrate is **not** a mirror of every GitHub repository notification. It separates:

1. **source occurrences** needed to reconstruct supported issue, pull-request/review, and Discussion conversation state; and
2. **attention-eligible community items**: externally authored, response-bearing occurrences that may need maintainer attention.

Stars/un-stars, forks, watches, and equivalent popularity telemetry are outside the community-event source families and cannot enter Bird View, counts, wake, or Task claim. Internal/rostered actions may update or resolve the state of an existing external item without minting new community attention. First-time versus trusted-repeat external status affects trust/projection, not basic eligibility. Bot eligibility must be an explicit ADR disposition and cannot be inferred from provider actor kind or trust tier.

Attention eligibility remains zero-authority: it does not assign work, enter LifecycleFrontier, or create a Task. Only the explicit canonical claim transition owns that promotion.

## Acceptance Criteria

- [ ] **AC1** — Output shape is fixed and bounded, with count, oldest occurrence time, coverage, and source watermark only.
- [ ] **AC2** — No title, body, excerpt, author prose, ranking score, or assignment hint is present.
- [ ] **AC3** — Every query/channel read is server-tenant- and source-scoped; cross-tenant count leakage tests pass.
- [ ] **AC4** — Claimed and explicitly ineligible items are excluded; seen items remain governed by the stated non-authority contract.
- [ ] **AC5** — Missing/degraded coverage is visible and never normalized to zero.
- [ ] **AC6** — Integration uses ADR 0035's projection writer contract or remains query-only until that owner ships; no parallel writer is created.
- [ ] **AC7** — No legacy lifecycle-state file, hook-time network read, global Neo roster, or all-resident broadcast is added.
- [ ] **AC8** — No stale-age, cadence, TTL, steward, or wake threshold is selected.

- [ ] **AC9** — Counts and oldest-age fields include only attention-eligible unclaimed items; internal/rostered context and excluded popularity telemetry leave both unchanged.
- [ ] **AC10** — A noise-flood fixture proves star/fork/watch volume cannot manufacture attention, while first-time and trusted-repeat external-human items remain countable.

## Out of Scope

Leased wake, Fleet UI, prose drill-down, ranking, Task claim, or ADR 0035 writer implementation.

## Avoided Traps

Do not treat zero as complete when coverage is degraded, let seen hide work, expose prose, or rebuild hook transport.

## Related

- Parent: neomjs/neo-agent-brain#106
- Source: Discussion #15139 OQ5/OQ7/OQ8/OQ10
- External owner: neomjs/neo-agent-brain#107 and ADR 0035
- Warning precedent: neomjs/neo#12850

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a

## Handoff Retrieval Hints

- `tenant community count only projection no prose`
- `ADR 0035 community attention channel`


## Creation Freshness

Creation duplicate sweep: immediately before filing at 2026-07-14T05:32:00.414Z, checked the latest 20 open issues and last 30 all-state A2A messages. The independent broader audit at 2026-07-14T05:13:00Z covered open and closed issues, pull requests, A2A, ADRs, and code; no equivalent owner or foreign claim existed.


## Timeline

- 2026-07-14T05:32:02Z @neo-gpt-euclid added the `enhancement` label
- 2026-07-14T05:32:02Z @neo-gpt-euclid added the `ai` label
- 2026-07-14T05:32:03Z @neo-gpt-euclid added the `architecture` label
- 2026-07-14T05:32:03Z @neo-gpt-euclid added the `security` label
- 2026-07-14T05:32:58Z @neo-gpt-euclid marked this issue as being blocked by #15152
- 2026-07-14T05:32:59Z @neo-gpt-euclid marked this issue as being blocked by #15153
- 2026-07-14T05:34:32Z @neo-gpt-euclid cross-referenced by #106
- 2026-07-22T11:43:28Z @neo-gpt-emmy cross-referenced by #15524
- 2026-08-24T11:39:12Z @neo-gpt-euclid marked this issue as blocking #3
- 2026-08-26T15:11:06Z @neo-gpt-euclid marked this issue as blocking #98
- 2026-08-26T15:11:43Z @neo-gpt-euclid marked this issue as blocking #100
- 2026-08-26T15:13:11Z @tobiu added the `enhancement` label
- 2026-08-26T15:13:11Z @tobiu added the `ai` label
- 2026-08-26T15:13:11Z @tobiu added the `architecture` label
- 2026-08-26T15:13:12Z @tobiu added the `security` label
- 2026-08-26T15:13:18Z @neo-gpt-euclid marked this issue as being blocked by #15152
- 2026-08-26T15:13:18Z @neo-gpt-euclid marked this issue as being blocked by #15153
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as being blocked by #102
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as blocking #100
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as blocking #3
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as blocking #98
- 2026-08-26T15:13:38Z @neo-gpt-euclid marked this issue as being blocked by #105
- 2026-08-26T15:14:31Z @tobiu added parent issue #106
- 2026-08-26T15:14:38Z @neo-gpt-euclid marked this issue as being blocked by #107
- 2026-09-19T23:01:07Z @neo-gpt-euclid cross-referenced by PR #392
- 2026-10-26T21:28:08Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-26T21:28:13Z

## Ticket Intake & Sharpening (`#101`)

### 1. Ticket Reality Classification (`ticket-intake §1.8`)
- **Ticket age:** `createdAt: 2026-07-14T05:32:00Z`, `updatedAt: 2026-08-26T15:13:10Z`
- **Bot stale-band:** `pre-stale` (`stale`: false, `no auto close`: false)
- **Epic-Review Pre-Requisite:** Satisfied by `neomjs/neo-agent-brain#106#issuecomment-6426058715` [QUARANTINED_URL: github.com] (`IC_kwDOUBzDFM8AAAABfwXf2w`, non-author Gemini Liveness Revalidation & `Greenlight`).
- **ADR Successor-Risk & Dependency Check:** `ADR 0036` (`learn/agentos/decisions/0036-durable-community-activity-authority.md`) and `ADR 0035` (`learn/agentos/decisions/0035-live-lane-awareness-composition.md`) are accepted on `dev`. Canonical Task binding (`mc_community_task_binding`) is implemented in `neomjs/neo-agent-brain#1307` [QUARANTINED_URL: github.com] (`#102`), so `#101` branches stacked on `agent/102-community-task-claim` to consume `mc_community_task_binding` directly.
- **Core-Idiom Pre-Flight (`§1.9.6`):** Verified `src/Neo.mjs`, `src/core/Base.mjs`, and `src/state/Provider.mjs` for `CommunityActivityService` (`Neo.setupClass` singleton extending `Neo.core.Base`).
- **Prescription Check (`§0`):**
  - `Prescription checked: ai/services/memory-core/CommunityActivityService.mjs — owns the concern` (`projectAttentionCounts` + `submitAttentionProjectionChannel`).
  - `ai/services/memory-core/hookProjectionLease.mjs` pins `'community-attention': 'community-attention.v1'` in `PROJECTION_CHANNEL_SCHEMAS` and `'community-attention': 'communityAttention'` in `PROJECTION_CHANNEL_SLOTS` so ADR 0035's existing `submitProjectionChannel` / `makeHookProjectionWriter` owner validates and composes the channel without any parallel file writer.
  - `ai/mcp/server/memory-core/communityActivityTool.mjs`, `ai/mcp/server/memory-core/toolService.mjs`, and `ai/mcp/server/memory-core/openapi.yaml` expose `get_community_activity_attention_counts` (`POST /community/activity/attention-counts`, `x-neo-tool-tier: read`).
- **Verdict:** `valid-as-written` (Positive ROI).

### 2. Concrete Contract Sharpening (`AC1`–`AC10`)
1. **Bounded Envelope (`community-attention.v1`, `AC1`, `AC2`, `AC5`, `AC8`):**
   - Returns `{schemaVersion: 'community-attention.v1', status: 'fresh' | 'empty' | 'missing' | 'degraded', notAuthority: true, count, oldestOccurredAt, sourceWatermark, coverage, sources, measurement}`.
   - Zero prose fields (`title`, `body`, `excerpt`, `actorId`, `url`), zero ranking/priority scores, zero assignment hints, and zero age/cadence/TTL/wake thresholds.
   - `status` is `'missing'` when the tenant has no registered sources (`sourceCount === 0`) and `'degraded'` when any scoped source or timestamp is incomplete/inactive/unsupported, ensuring missing or degraded coverage is never normalized to a healthy zero (`AC5`).
2. **Eligibility & Non-Authority `seen` vs. Canonical `claim` (`AC4`, `AC9`, `AC10`):**
   - Counts and `oldestOccurredAt` include only rows in `mc_community_observation` under the server-derived `tenantId` (`SourceRegistryService.resolveTenantId()`) where:
     - the owning `mc_source_registration` is `ACTIVE` and a supported GitHub repository (`provider = 'github'`, `canonical_provider_host = 'github.com'`, `resource_kind = 'repository'`);
     - `attention_disposition = 'eligible'` and `occurrence_kind IN (SUPPORTED_GITHUB_COMMUNITY_KINDS)`;
     - the entity has no recorded deletion tombstone (`absence = 'deleted'` with valid `deletion_evidence`);
     - no canonical claim binding exists in `mc_community_task_binding` for `(tenant_id, occurrence_identity)`.
   - Per-viewer `mc_community_seen` markers never reduce `count` or advance `oldestOccurredAt`.
3. **Monotonic Watermark & ADR 0035 Channel Integration (`AC6`, `AC7`):**
   - `sourceWatermark` encodes zero-padded `(admitted_sequence, claim_watermark, registration_watermark)` so batch admissions, canonical Task claims, and source lifecycle transitions advance the watermark monotonically under `submitProjectionChannel`'s lexicographical comparison while identical state replays idempotently.

---
Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cd90984f-706d-48de-8f9d-05a76ea2749c`. 🧭

- 2026-10-26T21:46:28Z @neo-gemini-maryam cross-referenced by PR #1308
- 2026-10-26T22:20:05Z @neo-gemini-maryam referenced in commit `27e4a32` - "feat(memory-core): ship query-only attention counts and bound watermark (#101)"
- 2026-10-26T22:32:08Z @neo-gemini-maryam cross-referenced by PR #1311
- 2026-10-26T22:38:31Z @tobiu referenced in commit `ba773d4` - "feat(memory-core): project bounded tenant community-attention counts (#101) (#1308)

* feat(memory-core): bind community events to canonical A2A Tasks (#102)

* feat(memory-core): project bounded tenant community-attention counts (#101)

* feat(memory-core): ship query-only attention counts and bound watermark (#101)"
- 2026-10-26T22:38:32Z @tobiu closed this issue
- 2026-10-26T23:31:08Z @neo-gemini-maryam cross-referenced by PR #1312
- 2026-10-26T23:34:23Z @neo-gemini-maryam cross-referenced by PR #1313

