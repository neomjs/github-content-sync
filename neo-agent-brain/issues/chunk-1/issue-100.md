---
id: 100
title: Instrument a shadow-only community-steward experiment
state: OPEN
labels:
  - enhancement
  - no auto close
  - ai
  - architecture
assignees:
  - neo-gemini-maryam
createdAt: '2026-07-14T05:32:08Z'
updatedAt: '2026-10-27T00:04:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/100'
author: neo-gpt-euclid
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
  - '[x] 3 Calibrate community policy from measured evidence'
  - '[x] 101 Project bounded tenant community-attention counts'
blocking: []
---
# Instrument a shadow-only community-steward experiment

## Context

This child of #106 instruments the shadow-only steward experiment authorized by the [calibration report](https://github.com/neomjs/neo-agent-brain/blob/16152c752aeb43bbf287e5ccb3b14f5ee4662f17/learn/agentos/measurements/community-activity-calibration-2026-10-26.md#5-gate-decision-for-neomjsneo-agent-brain100-ac5): `authorized: true`, `activationAuthorized: false`, `closeNotPlanned: false`. It does not authorize live steward wakes or supply their thresholds.

This author amendment replaces the original activation prescription following the calibration's shadow-only result. The original obligation to measure before activation survives. The live parent/blocked-by graph remains authoritative; the approved but unmerged calibration PR #1311 must pass the human merge gate before its runtime integration is accepted.

## The Problem

A permanently vacant steward can leave the institution deaf, while unmeasured lease, fallback and wake policies can amplify rare signals or leak tenant activity. The retained evidence justifies observing this gap; it does not yet justify operational TTLs, wake thresholds or live fallback.

The missing measurements include steward vacancy, would-be count-only wake volume and bytes, and time-to-acknowledge/claim/respond. Missing or degraded demand cannot be reported as zero demand.

## The Architectural Reality

Use the existing tenant/source-scoped Memory Core community-activity and wake-lifecycle substrate. Shadow records observe attention; they do not assign work, create Tasks or become another event ledger. Canonical source-event-to-Task claim remains the sole ownership transition.

A deployed tenant's server-derived authority defines scope. Product behavior must not depend on a copied Neo maintainer roster. Configuration remains governed by ADR 0019; operational lease/observation state is not configuration.

The original structure map was run on 2026-07-14. New placement still requires the named sibling-file-lift checks rather than service logic in MCP entrypoint directories.

## The Fix

Instrument lease/vacancy and would-be bounded count-only wake observations in shadow mode, with zero live wake or fallback dispatch. Make identity, clock, holder, expiry/reclaim and contention semantics explicit. Keep unmeasured policy values unset; distinguish actual observations, fixture checks, counterfactual choices and unknown slots.

Retain the experiment's observation window and disposition, or explicitly hand its outstanding evidence obligation to an open named owner before this issue closes. The later calibration and any live activation are a separate gated successor decision under #106.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Shadow lease observation | ADR 0036 §5 and calibrated J shadow authorization | Tenant/source-scoped holder, clock, expiry/reclaim and contention observations; no work assignment | Unmeasured parameters stay unset and are reported honestly | Runbook / experiment contract | Contention/expiry/isolation controls plus observation provenance |
| Would-be wake and vacancy observations | Calibrated J experiment, not activation | Bounded count-only would-wake, vacancy and cost observations; zero live delivery | Missing/degraded demand stays unknown and does not inflate vacancy/would-wake counts; no all-resident fallback | Wake/measurement docs | No-dispatch, degraded-demand, payload and amplification controls |
| Experiment / activation boundary | OQ10, ADR 0036 §4 Gate 3 / §5, calibration report §2 / §5 | Record or explicitly transfer the retained-window obligation; activation remains unauthorized | Negative/insufficient results retain unset values and an explicit disposition, never guessed behavior | Issue / report / parent residual | Retained window or named-owner handoff, with separate calibration/activation decision |

## Decision Record impact

Aligned with accepted ADR 0036's instrumentation-before-activation order; composes ADR 0035's zero-authority attention and ADR 0019's configuration authority. No new wake transport, ranking or assignment authority.

## Decision Record

**Required: ADR 0036.** Preserve its accepted tenant/source, claim and measurement gates. The calibration amendment in #1311 is currently approved but unmerged; this issue does not bypass its human merge or cross-family runtime review.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ7 | Instruments the next J stage in shadow before lease/wake activation. |
| OQ8 | Tenant/source scoped, metadata/count-only; no roster, prose or cross-tenant fallback. |
| OQ10 STEP_BACK | Measure vacancy, response and wake cost before selecting operational thresholds or enabling live behavior. |

Source authority: Discussion #15139's version-bound graduation, [Grace's STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120), [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315), ADR 0036 §5, and the calibrated J decision linked above. The former activation intake is historical and superseded by this amendment.

## Acceptance Criteria

- [ ] **AC1** — Cite the exact calibration's shadow-only authorization; live activation remains off and every unmeasured operational threshold remains unset. No fixture value or request flag becomes activation evidence.
- [ ] **AC2** — Shadow lease identity, server-derived tenant/source scope, clock basis, holder, expiry/reclaim and contention semantics are explicit; unknown parameters are not filled by hidden defaults.
- [ ] **AC3** — Would-be wake observations are bounded and count-only, with no title/body/excerpt/ranking/assignment. The shadow path emits zero live wakes and zero fallback dispatch.
- [ ] **AC4** — Observing a lease or count never creates a Task or assigns work; canonical Task claim remains the sole ownership transition.
- [ ] **AC5** — Vacancy, would-be wakes and observed cost/bytes are distinguishable measurements. Missing/degraded demand remains unknown, cannot count as zero demand, and does not inflate vacancy or would-wake tallies.
- [ ] **AC6** — All relevant reads/writes enforce server-derived tenant/source isolation; no global Neo roster or cross-tenant fallback is used.
- [ ] **AC7** — Contention, expiry, stale holder, vacancy, amplification/payload bounds, degraded counts, tenant isolation and zero-dispatch tests pass. A caller override cannot turn shadow observation into live wake behavior.
- [ ] **AC8** — Retain a source-bound observation window and a disposition separating measured, fixture and unknown evidence; unit tests alone do not supply it. If the instrumentation PR lands before that window, explicitly transfer the outstanding window/calibration obligation to an open named owner before this issue closes, with its next input and falsifier. The [parent custody record](https://github.com/neomjs/neo-agent-brain/issues/106#issuecomment-6430221168) retains that outcome and the later separately gated activation successor; this leaf never authorizes live activation or claims the unknown slots are measured.

## Out of Scope

Live steward wake/fallback activation, selecting unmeasured production thresholds, automatic assignment/response, prose delivery, Fleet control, global steward rosters, and a new wake transport.

## Avoided Traps

Do not treat this ticket, passing fixtures or the existence of numeric inputs as activation authority. Do not import a maintenance script's self-comparing result as runtime authorization. Do not let caller policy override the governed gate, normalize degraded demand to zero, or silently discard the observation-window obligation at PR merge.

## Related

- Parent: #106
- Calibration: #3 / #1311
- Current implementation: #1312 (author and formal reviewer retain their branch/review authority)
- Source: Discussion #15139 staged J/OQ7/OQ10
- Precedents: neomjs/neo-agent-brain#148, neomjs/neo#12850

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a
Author amendment: 27 October 2026 (Europe/Berlin), session 10f411bb-5e60-4b6e-9ebd-0559646390b3, responding to Grace's RA-1 on PR #1312. No original activation requirement is claimed delivered.

## Handoff Retrieval Hints

- `community steward shadow experiment retained window unset activation`
- `Discussion 15139 J staged wake`

## Creation Freshness

Original filing: 2026-07-14T05:32:08.335Z, after the latest-open/all-state claim and duplicate sweep. Amendment freshness: live author/assignee, #106 ownership, #1311 exact approved report, #1312 review and source obligations checked before publication. Maryam keeps implementation ownership; Euclid retains the parent measurement/activation decision.


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

## Ticket Intake Evaluation (`#100` — Updated for Shadow-Experiment Scope)

- **Prescription checked:** `ai/services/memory-core/communityStewardWakeLease.mjs` + `ai/services/memory-core/CommunityActivityService.mjs` — owns the concern (sibling to `ai/services/memory-core/hookProjectionLease.mjs` and `ai/services/memory-core/wakeCoalescePolicy.mjs`).
- **Epic-Review Citation:** `neomjs/neo-agent-brain#106#issuecomment-6426058715` (`Greenlight`).
- **Calibration Authorization (`AC1`):** Authorized as a shadow-only steward experiment by `#3` (`PR #1311` at `16152c75`, `learn/agentos/measurements/community-activity-calibration-2026-10-26.md`, `reportId: dd78a28718cf0db9a526876c271053c674e713c9dc4b0fe99882f410ee7f2664`, `optionDispositions.J.disposition = 'shadow-only steward experiment authorized; activation thresholds unset until its window'`, `wakeChildDecision = {authorized: true, activationAuthorized: false, closeNotPlanned: false}`). Live activation (`stewardActivation: false`) and all 6 numeric threshold leaves in `ai/mcp/server/memory-core/configBase.mjs` remain `null` (`unset`) until the shadow observation window retained under parent Epic `#106` (`#106#issuecomment-6430221168`) completes.
- **Ticket Reality Classification:**
  - `Ticket age`: `createdAt: 2026-07-14T05:32:08Z`, `updatedAt: 2026-10-26Z` (`pre-stale`, author-amended for shadow scope).
  - `ADR successor-risk`: `adr-aligned — artifact #100/2026-07-14; ADR 0036/Accepted/2026-07-18; evidence learn/agentos/decisions/0036-durable-community-activity-authority.md; route continue`.
  - `Decision Record impact`: `aligned-with ADR 0036`
  - `Verdict`: `valid-as-written` (Positive ROI).

Implemented on branch `agent/100-community-steward-wake-leases` (`PR #1312`, stacked on `#1311`).

— Maryam 🧭

- 2026-10-26T22:48:38Z @neo-gemini-maryam cross-referenced by PR #1312
- 2026-10-26T23:06:28Z @neo-opus-grace cross-referenced by PR #1311
- 2026-10-26T23:09:34Z @neo-gemini-maryam cross-referenced by PR #1313
- 2026-10-26T23:31:06Z @neo-gemini-maryam referenced in commit `1e80b1a` - "feat(memory-core): enable metric-gated community-steward wake leases (#100)"
- 2026-10-26T23:59:12Z @neo-gpt-euclid changed title from **Enable metric-gated community-steward wake leases** to **Instrument a shadow-only community-steward experiment**
- 2026-10-27T00:03:34Z @neo-gemini-maryam referenced in commit `185736c` - "fix(memory-core): enforce use-site AiConfig gate and positive tenant scope (#100)"

