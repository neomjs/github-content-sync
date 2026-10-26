---
id: 100
title: Enable metric-gated community-steward wake leases
state: OPEN
labels:
  - enhancement
  - no auto close
  - ai
  - architecture
assignees:
  - neo-gemini-maryam
createdAt: '2026-07-14T05:32:08Z'
updatedAt: '2026-10-26T22:34:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/100'
author: neo-gpt-euclid
commentsCount: 1
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy:
  - '[ ] 3 Calibrate community policy from measured evidence'
  - '[x] 101 Project bounded tenant community-attention counts'
blocking: []
---
# Enable metric-gated community-steward wake leases

## Context

This conditional child preserves staged option J now so no later agent invents wake behavior from the Epic body. It is deliberately not code-ready until the calibration leaf supplies acceptable thresholds; if evidence rejects activation, it closes not planned with that evidence.

This is one fully closeable PR leaf under Epic neomjs/neo-agent-brain#106. The live parent-child and blocked-by graph is authoritative; this body owns only this leaf's contract.

## The Problem

All-resident wake duplicates rare signals and can leak tenant activity; a permanent steward can leave the institution deaf when vacant. A lease without measured TTL/amplification bounds becomes hidden assignment and wake-storm authority.

## The Architectural Reality

Wake coordination belongs in the existing Orchestrator/wake lifecycle substrate, consuming the count-only projection and canonical claim state. The lease is tenant-scoped attention routing, never ownership. It must not use the global Neo maintainer roster in external deployments.

The Agent OS structure map was run on 2026-07-14. New service/script/test placement must use the named sibling-file-lift fast paths; no service logic moves into MCP server entrypoint directories.

## The Fix

Only after calibration approval, implement a tenant/source-scoped TTL-bound steward lease, coalesced count-only wake, measured vacancy fallback, explicit expiry/reclaim, and amplification caps. Keep prose, ranking, assignment, and cross-tenant fallback out of the path.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Steward lease | OQ7 J-stage2 | Tenant-scoped TTL lease routes attention without assigning work | Vacancy/expiry follows measured fallback | Runbook | Contention/expiry tests |
| Coalesced wake | Calibration evidence | Count-only bounded wake under measured amplification cap | Degraded/missing count suppresses enrichment, not policy | Wake docs | Amplification and failure tests |
| Activation gate | OQ10 | No implementation/enablement before calibration result | Rejected evidence closes not planned | Issue/ADR link | Review verifies cited report |

## Decision Record impact

Depends on ADR 0036, count-only projection, and the calibration decision; composes ADR 0035 without adding ranking authority.

## Decision Record

**Required: ADR 0036.** This leaf is not code-ready until the ADR-0036 child of neomjs/neo-agent-brain#106 is accepted at the human merge gate.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ7 | Implements only the second, leased-wake stage of J. |
| OQ8 | Tenant/source scoped, metadata/count-only, no roster or prose leakage. |
| OQ10 STEP_BACK | Every lease, vacancy, coalescing, and wake threshold cites measured evidence. |

Source authority: Discussion #15139 body at the version-bound graduation anchor plus Grace's [STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120) and [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315).

## Acceptance Criteria

- [ ] **AC1** — The calibration artifact explicitly authorizes activation and supplies every numeric threshold.
- [ ] **AC2** — Lease identity, tenant/source scope, TTL, expiry, reclaim, and contention semantics are explicit.
- [ ] **AC3** — Wake payload is count-only and bounded; it contains no title/body/excerpt/ranking/assignment.
- [ ] **AC4** — The lease routes attention only; canonical Task claim remains the sole ownership transition.
- [ ] **AC5** — Vacancy fallback is measured, bounded, and cannot become all-resident broadcast.
- [ ] **AC6** — No global Neo roster or cross-tenant fallback is used.
- [ ] **AC7** — Contention, expiry, stale holder, vacancy, amplification cap, degraded counts, and tenant isolation tests pass.
- [ ] **AC8** — If activation evidence is negative or insufficient, close this issue as not planned with the calibration citation instead of shipping guessed behavior.

## Out of Scope

Automatic assignment/response, prose delivery, Fleet control, global steward roster, unmeasured thresholds, or a new wake transport.

## Avoided Traps

Do not interpret the existence of this ticket as activation authority, invent TTLs, broadcast to all residents, or let leases create Tasks.

## Related

- Parent: neomjs/neo-agent-brain#106
- Source: Discussion #15139 staged J/OQ7/OQ10
- Precedents: neomjs/neo-agent-brain#148, neomjs/neo#12850

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a

## Handoff Retrieval Hints

- `community steward lease coalesced count wake metric gated`
- `Discussion 15139 J staged wake`


## Creation Freshness

Creation duplicate sweep: immediately before filing at 2026-07-14T05:32:08.335Z, checked the latest 20 open issues and last 30 all-state A2A messages. The independent broader audit at 2026-07-14T05:13:00Z covered open and closed issues, pull requests, A2A, ADRs, and code; no equivalent owner or foreign claim existed.

## Timeline

- 2026-07-14T05:32:09Z @neo-gpt-euclid added the `enhancement` label
- 2026-07-14T05:32:09Z @neo-gpt-euclid added the `no auto close` label
- 2026-07-14T05:32:10Z @neo-gpt-euclid added the `ai` label
- 2026-07-14T05:32:10Z @neo-gpt-euclid added the `architecture` label
- 2026-07-14T05:34:32Z @neo-gpt-euclid cross-referenced by #106
- 2026-08-24T11:39:13Z @neo-gpt-euclid marked this issue as being blocked by #3
- 2026-08-26T15:11:43Z @neo-gpt-euclid marked this issue as being blocked by #3
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as being blocked by #101
- 2026-08-26T15:14:31Z @tobiu added parent issue #106
- 2026-09-19T23:01:07Z @neo-gpt-euclid cross-referenced by PR #392
- 2026-10-26T22:34:49Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-26T22:34:54Z

## Ticket Intake Evaluation (`#100`)

- **Prescription checked:** `ai/services/memory-core/communityStewardWakeLease.mjs` + `ai/services/memory-core/CommunityActivityService.mjs` — owns the concern (sibling to `ai/services/memory-core/hookProjectionLease.mjs` and `ai/services/memory-core/wakeCoalescePolicy.mjs`).
- **Epic-Review Citation:** `neomjs/neo-agent-brain#106#issuecomment-6426058715` [QUARANTINED_URL: github.com] (`Greenlight`).
- **Calibration Authorization (`AC1`):** Authorized by `#3` (`PR #1311`, `learn/agentos/measurements/community-activity-calibration-2026-10-26.md`, `reportId: 62f4d25f6ce7a5b6b28198f716aed78fc28c14c9fd94e9919e1322ea718e863d`, `optionDispositions.J.disposition = 'activate-gated'`, `wakeChildDecision.authorized = true`) with calibrated numeric bounds:
  - `stewardLeaseTtlMs: 1_590_000` (`[795_000, 3_600_000]`)
  - `wakeCoalesceWindowMs: 397_500` (`[60_000, 900_000]`)
  - `vacancyFallbackCooldownMs: 795_000` (`[397_500, 3_600_000]`)
  - `minUnclaimedEligibleCount: 1` (`[1, 10]`)
  - `wakeAmplificationCap: 1` (`[1, 1]`)
  - `maxWakePayloadBytes: 1_024` (`[256, 2_048]`)
- **Ticket Reality Classification:**
  - `Ticket age`: `createdAt: 2026-07-14T05:32:08Z`, `updatedAt: 2026-10-26Z` (`pre-stale`).
  - `ADR successor-risk`: `adr-aligned — artifact #100/2026-07-14; ADR 0036/Accepted/2026-07-18; evidence learn/agentos/decisions/0036-durable-community-activity-authority.md; route continue`.
  - `Decision Record impact`: `aligned-with ADR 0036`
  - `Verdict`: `valid-as-written` (Positive ROI).

Claiming `#100` on branch `agent/100-community-steward-wake-leases` (stacked on `#1311`).

— Maryam 🧭

- 2026-10-26T22:48:38Z @neo-gemini-maryam cross-referenced by PR #1312
- 2026-10-26T23:06:28Z @neo-opus-grace cross-referenced by PR #1311
- 2026-10-26T23:09:34Z @neo-gemini-maryam cross-referenced by PR #1313
- 2026-10-26T23:31:06Z @neo-gemini-maryam referenced in commit `1e80b1a` - "feat(memory-core): enable metric-gated community-steward wake leases (#100)"

