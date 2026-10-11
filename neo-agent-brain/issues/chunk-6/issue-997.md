---
id: 997
title: Retain lifecycle obligations when their clock is unavailable
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-10-11T06:47:48Z'
updatedAt: '2026-10-11T07:09:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/997'
author: neo-gpt
commentsCount: 1
parentIssue: 107
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
# Retain lifecycle obligations when their clock is unavailable

## Context

The clock contract in ADR 0035 §2.3.1 merged through #996 as `4dd2054a`. #107's source audit established that runtime adoption must update the existing source/admission → envelope → publication → shared-hook path coherently. The accepted record retains real response obligations when their sources explicitly cannot date them.

## The Problem

Current v1 requires a dated `actionableSince` and labels every rendered row “actionable since”. `lifecycleAdmission.mjs` still clamps PR clocks to commit creation and derives reviewer routing from evergreen green checks; neither proves current PR head activation or uninterrupted reviewability. A builder-only nullable change would also fail publication, whose lifecycle slot is pinned to v1.

Design authority: ADR 0035 §2.3.1: “Eligibility and clock knowledge are separate” and “Adopt the incompatible shape coherently.”

## The Architectural Reality

Existing pure graph owners `lifecycleAdmission.mjs`, `lifecycleFrontier.mjs` and `produceLifecycleFrontier.mjs` own the admitted rows and envelope. `hookProjectionLease.mjs:74` owns the publication schema pin; `hookProjectionReader.mjs:364–407` validates and renders the shared hook overlay. Source facts remain injected; a production GitHub adapter, target registry and launcher attester are separate #107 obligations.

## The Fix

Replace the lifecycle shape with v2 in one PR. Consume explicit source-owned clock declarations, preserve head binding and eligibility, remove both `headCommittedAt` shortcuts and evergreen fallback, validate nullable clock knowledge, sort dated rows before unavailable rows within each unchanged stage, advance the publication pin and render each basis truthfully. Use existing canonical specs; no new service or adapter.

## Contract Ledger

| Target surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Admission normalization | Source clock declaration + current PR head | Preserve explicit transition/observation/unavailable declarations; all existing admission predicates still apply | Missing, malformed or wrong-head declaration is an error, never inferred unavailable | ADR 0035 §2.3.1 | Exact current-head, re-entry, zero-required and missing-declaration controls |
| Frontier builder/validator/order | ADR 0035 v2 | Null only with unavailable basis and bounded reason; dated rows retain valid timestamp/basis; stable ordering | Refuse invalid v2 and v1 | ADR 0035 §2.3 | Paired valid/invalid and stage-order controls |
| Publication lifecycle slot | `PROJECTION_CHANNEL_SCHEMAS` | Admit v2 into the existing lifecycle channel | v1/mismatched schema refused; sibling route preserved | ADR 0035 §2.6 | Real submission/publication composition |
| Shared hook rendering | Validated v2 clock basis | Transition: actionable since; observation: observed actionable at; unavailable: bounded reason; capture-time as-of retained | Existing stale/foreign/bare-policy behavior | ADR 0035 §2.3.1 | Published envelope → shared reader, including refusal controls |

Prescription checked independently: the six existing owners own this concern; no config/daemon/adapter replacement solves the incompatible clock contract. The non-author reader checked the merged record and production modules before edits.

Normalized source shape: own-PR fields retain `repairActionableSince` / `reviewableSince`, their `*HeadSha` provenance and companion `*ClockBasis` / `*ClockReason`. Review-request objects carry per-target `login`, `actionableSince`, `headSha`, `clockBasis`, `clockReason`; raw `requestedAt` remains source-event metadata; normalization retains the declarations in `reviewRequestedByTarget`. Task/message source records declare basis/reason beside their existing clock. A controlled unavailable reason is one nonempty line of at most 240 characters. No current-head proof is synthesized from the enclosing PR.

Decision Record impact: aligned-with ADR 0035, implementing its merged clock amendment; no new ADR or changed authority.

## Acceptance Criteria

- [ ] A valid explicitly unavailable clock retains an otherwise admitted obligation; malformed/missing declarations and wrong-head provenance fail closed without becoming empty or invented dates.
- [ ] PR normalization removes commit-time floor and zero-required-check commit fallback; evergreen checks cannot date uninterrupted reviewability. Exact-head declarations, later head activation, same-head re-entry, zero-required checks and target-specific requests are covered.
- [ ] Builder and validator agree on v2 clock fields and order: fixed stage rank, dated before unavailable, oldest dated first and stable id ties. Clock availability does not alter envelope freshness or eligibility.
- [ ] Actual submission → fenced publication → shared reader accepts an unavailable row and a dated positive; v1 is rejected and sibling channels are preserved.
- [ ] Shared rendering distinguishes all three bases, preserves as-of and existing stale/foreign/refusal and pure Claude/Codex append behavior.
- [ ] Canonical focused and owning checks pass. The PR records source/fixture evidence separately from future production-adapter and installed acceptance.

## Out of Scope

Production GitHub lifecycle adapter; tenant/source registration or instance attestation; host-root publication installation; two live producer cycles/multi-resident acceptance; offline exposure/outcome capture. #107 remains open.

## Avoided Traps

No missing-field coercion, commit-time head-entry guess, discovery-pulse aging, dual-read v1 migration or hook-time network read. This is one coherent adoption, not separate incompatible consumer PRs.

## Related

Parent #107; record #995 / #996; existing acceptance map https://github.com/neomjs/neo-agent-brain/issues/107#issuecomment-6104013184.

## Creation Evidence

Structure map executed: graph pure admission/frontier siblings, Memory Core projection schema and shared lifecycle reader are existing owners. No new `.mjs` placement.
Live latest-open sweep: latest 20 open issues and “lifecycle clock” search checked at creation; only the closed record #995 overlaps, no implementation leaf.
Own-assignment sweep: six open; #107 is the retained outcome, the other five concern unrelated deployment/community work.
MC sweep: three framings recovered session `2d8feac7-c60d-4883-8059-37b6e148768b` and the accepted clock record; no competing implementation surfaced.
A2A scope sweep: recent all-state messages checked immediately before creation; no clock-adoption claim.

Origin Session ID: 2d8feac7-c60d-4883-8059-37b6e148768b
Retrieval Hint: "107 unavailable lifecycle clock commit creation evergreen green v2 coherent publication reader"


## Timeline

- 2026-10-11T06:47:49Z @neo-gpt assigned to @neo-gpt
- 2026-10-11T06:47:50Z @neo-gpt added the `enhancement` label
- 2026-10-11T06:47:50Z @neo-gpt added the `ai` label
- 2026-10-11T06:47:50Z @neo-gpt added the `architecture` label
- 2026-10-11T06:47:51Z @neo-gpt added the `agent-os` label
- 2026-10-11T06:48:08Z @neo-gpt added parent issue #107
### @neo-gpt - 2026-10-11T06:50:28Z

Prescription checked independently before edits: lifecycleAdmission/frontier/producer, hookProjectionLease's channel pin and shared hookProjectionReader own this adoption; no new service or upstream adapter. Normalized PR declarations retain stage fields+head provenance with companion ClockBasis/ClockReason; review requests carry per-target requestedAt/headSha/clockBasis/clockReason. Controlled unavailable reason: one nonempty line, ≤240 characters. Root retains architecture/integration. Native pre-brief is not yet ingested; live ticket, merged ADR and session prior-art are the authority. Branch codex/997-lifecycle-clock-adoption from merged4dd2054a.

- 2026-10-11T07:07:39Z @neo-gpt cross-referenced by PR #998
- 2026-10-11T07:10:12Z @neo-gpt cross-referenced by #107

