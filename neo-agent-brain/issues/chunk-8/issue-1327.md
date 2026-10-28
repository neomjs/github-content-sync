---
id: 1327
title: Define matrix-bound community coverage qualification
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-27T23:30:10Z'
updatedAt: '2026-10-27T23:55:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1327'
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
closedAt: '2026-10-27T23:55:26Z'
---
# Define matrix-bound community coverage qualification

## Context

The real tenant/source shadow-window outcome remains owned by #106. [Discussion `20161`](https://github.com/orgs/neomjs/discussions/20161), body anchor `2026-10-27T23:23:23Z`, converged on matrix-bound acquisition completeness plus explicit provider limitations and bounded conclusions (B1 + D). It retains the original late-reply-after-closure falsifier from graduated `15139`.

The existing ADR 0036 is Accepted (2026-07-18, PR `15473`). Its H disposition says unsupported revisions/families remain coverage gaps. Production PR and Discussion reconcilers therefore retain three permanent gaps each, while the count reader requires all three families complete/zero-gap. A semantic qualification change must amend that accepted authority explicitly.

This is the **one-PR decision-record leaf**: amend the existing ADR, preserving its identity and original provenance. Runtime implementation, accepted deployment, real source acquisition/window, and independent calibration remain on #106. Closing this leaf cannot close that parent outcome.

## The Problem

An exhaustive supported traversal and missing provider history currently share one degradation result. A synthetic complete fixture cannot establish the actual producer/consumer contract. At Brain `06f5574e`, a pure real-compiler probe also shows the hosted schema refuses collector `{axis,reason}` gaps while the neutral envelope validator accepts the same 634-byte incomplete envelope. That interoperability repair is independently needed; it must not silently choose qualification semantics.

Peer falsifiers ruled out a closed-container bucket and an unknown-closer waiver. The latter would retain an old resolved close after a later open snapshot because admitted history is append-only and the adapter emits no Discussion reopen. The original post-close replies must remain eligible.

## The Architectural Reality

Design authority: ADR 0036 §2.1: reconciliation is exhaustive **within the declared matrix**; §2.4: connectors own acquisition/interpretation and Memory Core owns neutral validation/history; §2.6 and §5: counts/wakes remain zero-authority and measurement-gated.

The amendment touches `learn/agentos/decisions/0036-durable-community-activity-authority.md` at its existing canonical home. No new ADR number, decision tree, service, configuration key, generator or runtime path is required by this ticket. The relevant source seams (unchanged from `06f5574e` at current `40bb74cc`) are:
- adapter constants and completeness in `githubPullRequestReconciliation.mjs:9–21,628,827–831` and `githubDiscussionReconciliation.mjs:9–12,136,232–237`;
- coverage rebuild in `community/assembleIssueBatch.mjs:65–75`;
- canonical digest/validation in `communityBatchContract.mjs:235–245,297–323,381–400`;
- persisted coverage receipts in `CommunityBatchAdmissionService.mjs:129–145,347–358`;
- three-family read in `CommunityActivityService.mjs:376–398`;
- shared evaluator early coverage exit in `communityStewardWakeLease.mjs:569–578`.

## The Fix

Amend §2.1, §2.4, §2.6, H in §3, and the H revalidation trigger in §6. Record the new amendment provenance and retained implementation conditions without rewriting the original graduation history.

Specify:
1. Separate acquisition/access/verification gaps from declared provider-history limitations. Only a recognized versioned matrix with the exact required declaration set can qualify supported acquisition. Missing/extra/duplicate/wrong-family declarations and unknown tuples fail closed.
2. Persist the declaration/generation binding inside coverage evidence read from receipts. Top-level transient batch versions alone are insufficient. Preserve historical JSON, digest and exact-replay semantics; old/mixed generations are never promoted into qualified completeness.
3. Keep all three expected families, tenant/source/epoch fencing, current actor eligibility and internal/rostered closer predicates. Count is an upper bound within the eligible admitted population, not a universal exact-demand/recall claim.
4. Historical unattributed-close telemetry is labelled “latest admitted closure; reopen not observable.” Post-close activity stays eligible; no synthetic reopen or claim-for-cleanup is introduced. Unknown-actor telemetry counts actual exclusions, with response-bearing subsets unknown without independently bound policy.
5. Qualified zero demand can report observed lease vacancy and false would-wake through the shared evaluator; missing/degraded evidence cannot manufacture those values.
6. Local/hosted gap-item validation uses one neutral definition and a derived/drift-checked OpenAPI component. Assembly and every strict public response block preserve the declarations/measurement fields.
7. Main-window source is Neo (stable ID `220867748`) with all families enabled and corpus evidence of external activity. The known metadata implies 399 initial root pages before child/edit/verification/census reads. Total acquisition cost and provisioning/credential authority remain operator-owned. The cheaper empty control is not substituted for the full goal.

## Contract Ledger

| Target | Authority | Required meaning | Failure / fallback | Evidence |
|---|---|---|---|---|
| Acquisition completeness / provider limitations | ADR 0036 §2.1/§2.4; D20161 B1 | Versioned exact declaration plus zero real acquisition gaps | Unknown/missing/wrong declaration remains unqualified | Source constants, real compiler/envelope probe, negative controls required downstream |
| Persisted coverage generation | Receipt coverage JSON and canonical digest | Readable binding, immutable historical interpretation | Legacy complete-looking or mixed family receipts cannot qualify | STEP_BACK seam 2a |
| Count/empty/vacancy conclusions | §2.6; existing shared evaluator | Named admitted population, qualified-empty observed values | Missing/degraded values stay unknown | STEP_BACK seam 2b |
| Hosted/assembly/public shape | Neutral batch contract and OpenAPI | One gap-item definition; lossless declarations | Closed-key/authority/prose refusals remain | STEP_BACK seam 2c |
| Real window / activation | §4/§5 and #106 | Real retained source/store provenance and independent disposition | Activation false, thresholds unset; source approval separate | #106 custody and D20161 main-source disposition |

## Decision Record impact

**Amends ADR 0036; retains ADR 0015, ADR 0019 and ADR 0035 boundaries.** No AiConfig work or new deployment authority.

ADR successor-risk: **adr-amendment-required** — D20161 (2026-10-28 local date) postdates Accepted ADR 0036 (2026-07-18); the changed meaning of unsupported coverage must be accepted through an explicit amendment before qualification-changing runtime code is merge-ready.

## Decision Record

**Required: amend existing ADR 0036, first in merge order.** This docs-only PR is the authority transition; later runtime work must cite its human-merged revision.

## Acceptance Criteria

- [ ] ADR text consistently states matrix-bound acquisition completeness, exact persisted declaration/generation binding, failure cases, and immutable history/replay boundaries across all affected sections.
- [ ] Count/empty semantics, preserved closer policy, honest historical/unknown-actor metrics, zero-dispatch/activation limits and source/cost authority match the current D20161 fold.
- [ ] The three STEP_BACK partials (persisted generation visibility; qualified-empty vacancy/false would-wake; assembly plus strict response propagation) are explicit mandatory downstream criteria.
- [ ] Original ADR identity/provenance and unrelated authority boundaries remain intact; no code, schema migration, operational threshold, provisioning or activation is introduced.
- [ ] Links/decision-record checks pass and independent current-head review confirms fidelity. Human merge accepts this amendment; runtime and the real window remain open on #106.

## Out of Scope

Runtime implementation, source lifecycle or admission, provider acquisition, credentials, deployment, Task/lease/dispatch mutation, live activation, and claiming corpus/unit evidence as the real window. No family-selector API, organization audit-log grant, closer waiver, synthetic Discussion reopen, new epic or rewritten historical receipt.

## Avoided Traps

A same-head fixture result is not producer evidence; a declaration allowlist without mandatory-set binding permits omission; current snapshots do not erase admitted close history; and choosing a predictable empty source would avoid the goal's wake-cost population.

## Signal Ledger

- GPT author family: [AUTHOR_SIGNAL](https://github.com/neomjs/neo/discussions/20161#discussioncomment-19229945), body `2026-10-27T23:23:23Z`; current membership active/family `gpt`.
- Claude non-author family: [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/20161#discussioncomment-19229948), same body anchor; current membership active/family `claude`.
- [Maryam's STEP_BACK](https://github.com/neomjs/neo/discussions/20161#discussioncomment-19229871): seven passes and one partial with three explicit criteria. The source-density delta is acknowledged by Ada's same-anchor approval; no conditional substantive blocker remains. Maryam's review is retained without inventing family credit from a null family projection.

## Unresolved Dissent

None at the selected contract anchor. Three non-blocking implementation partials are preserved above. New material falsifiers reopen their delta.

## Unresolved Liveness

No liveness gap is inferred from silence. Grace is at a session handover and her invitation was explicitly rerouted. Maryam is active, but the read projection reports family unavailable; her contribution is not counted toward family quorum. The two required known active families have explicit signals.

## Discussion Criteria Mapping

OQ1/OQ4 → named population and bounded conclusions; OQ2 → exact declaration binding and real-gap refusal; OQ3 → persisted generation/history/replay; OQ5 → one neutral gap definition and local/hosted parity; OQ6 → strict closer and post-close/reopen limits. Source and STEP_BACK partials map to the mandatory downstream criteria. Runtime/window completion stays with #106.

## Related

Parent: #106
Source: [Discussion `20161`](https://github.com/orgs/neomjs/discussions/20161)
Baseline observer: [PR `1322`](https://github.com/neomjs/neo-agent-brain/pull/1322)

Origin Session ID: 607243d1-f786-4b5c-b0a2-028defcc0310
Retrieval Hint: `D20161 B1 bounded D persisted coverage declaration qualified empty strict closer`
Retrieval Hint: `HostedCommunityCoverage axis reason providerLimitations source-bound window ADR0036`

Creation receipts: live latest20open (newest1325) and all-state A2A30 inspected immediately before create; no competing coverage/ADR claim. MC problem-noun sweep returned noisy historical hits, not a competing coverage disposition; the live D20161/ADR/source are authority. Own-assignment sweep:6open,106 is the owning parent and107 the preserved zero-authority adjacency (bodies read). Agent OS structure-map --files --loc exited0; the existing ADR0036 path is canonical, no new directory/MJS role. Exact all-state coverage search and341Discussion-title inventory were retained during D20161 intake. Labels confirmed live. No epic is created.

Prescription checked: learn/agentos/decisions/0036-durable-community-activity-authority.md — owns the accepted semantics being amended; non-author planner checks are Maryam's D20161 authority sweep and Ada's current-contract approval. Config, transport or a caller guard cannot override that ADR. Same-session intake otherwise uses the self-authored carve.



## Timeline

- 2026-10-27T23:30:10Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-27T23:30:12Z @neo-gpt-euclid added the `documentation` label
- 2026-10-27T23:30:12Z @neo-gpt-euclid added the `enhancement` label
- 2026-10-27T23:30:12Z @neo-gpt-euclid added the `ai` label
- 2026-10-27T23:30:12Z @neo-gpt-euclid added the `architecture` label
- 2026-10-27T23:30:56Z @neo-gpt-euclid added parent issue #106
- 2026-10-27T23:41:42Z @neo-gpt-euclid cross-referenced by PR #1328
- 2026-10-27T23:45:20Z @neo-gpt-euclid cross-referenced by #106
- 2026-10-27T23:55:26Z @tobiu referenced in commit `d02bdd7` - "docs(agentos): define matrix-bound community coverage (#1327) (#1328)"
- 2026-10-27T23:55:26Z @tobiu closed this issue
- 2026-10-28T00:08:34Z @neo-gpt-euclid cross-referenced by #1329

