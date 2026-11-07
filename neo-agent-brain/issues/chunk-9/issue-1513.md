---
id: 1513
title: Use admitted families for routing and publication checks
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-11-06T23:06:55Z'
updatedAt: '2026-11-06T23:06:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1513'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: 1296
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1511 Wire the protected family service through Memory Core'
blocking: []
---
# Use admitted families for routing and publication checks

## Context and problem
Mailbox aliases, wake/presence filters and two content-publication family checks still use roster/flat graph fields. An admitted currentSeatId alone cannot prove which graph identity a provider subject routes to.

**Blocked on source, not a policy guess:** Institution [D825](https://github.com/neomjs/neo-agent-institution/discussions/825#discussioncomment-19426881) still has its old-login/rename alias disposition open. This leaf consumes the resulting implemented protected subject→graph-identity relation; it does not graduate that design, pick its unresolved policy or create a substitute carrier. Add the concrete relation-delivery issue as a native blocker once that owner files it. Until then, this body records the unresolved external gate and is not claimable for implementation.

## Architectural reality and intended change
Use the MC qualified family snapshot and the shared request-binding relation at the existing consumers: MailboxService family alias expansion and attestPublishRevision; WakeSubscriptionService family routing/filtering and corresponding presence reader; and Fleet CONTENT publishExecutor's production resolveIdentityFamily composition. The last two publication checks are distinct from #1297, which publishes FAMILY ACTS through its durable outbox/signer.

## Contract Ledger
| Surface | Authority | Behavior and refusal | Evidence / docs |
| --- | --- | --- | --- |
| Family aliases | Existing mailbox eligibility/alias policy + protected binding | Current admitted family; unchanged zero/multiple-match refusal | Real mailbox recipient paths, rename/new-subject controls |
| Wake/presence | Existing wake eligibility + qualified snapshot | Current classification, missing/ambiguous relation nonqualifying | Real family-filtered dispatch/presence controls |
| Content-publication family checks | Existing MC attestation and Fleet publish policy | Both use admitted families without changing publication permission | Actual attestPublishRevision and injected production resolver paths |

## Acceptance Criteria
- [ ] The external D825 relation is graduated AND implemented with its own protection, explicit missing/ambiguous outcomes, rename continuity and same-login/new-provider-ID refusal. The public ticket/PR names its concrete delivered source and native dependency. No reliance on currentSeatId or ordinary providerUserId properties as authority.
- [ ] Mailbox family aliases consume one coherent admitted current-family observation per resolution. Preserve eligibility, exact family vocabulary and zero/multiple-match refusals. No static identityRoots/modelFamily/family fallback.
- [ ] Wake and presence family filters consume that same qualified source with the existing activity/participation/eligibility semantics. Unknown/unavailable is not an empty population or permission to route elsewhere.
- [ ] BOTH MailboxService.attestPublishRevision and the Fleet content publishExecutor production resolveIdentityFamily injection use admitted families. Preserve the accepted content-publication policy; a helper fixture or #1297 family-act publisher alone is insufficient.
- [ ] Integrated recipient/publication controls cover unconfirmed/ambiguous/empty histories, family changes/corrections, same-subject rename, new subject reusing a login, stale/reconnected plane observations, and missing source. No caller chooses a routing identity through mutable login alone.
- [ ] Named consumer census has no runtime static-root/flat-field fallback in this scope. Actual production composition/refusal tests and Brain preflight/CI pass; docs expose remedies. The separate installed #414 journey remains post-merge.

## Scope, mapping and dependencies
Blocked by #1511 and the independently protected D825 relation. Own original AC-11 and routing/publication parts of AC-9/12. No new alias policy, request-binding implementation, broad #875 trust/display cleanup, #112 seed-era cleanup, private-key custody or #1297 family-act outbox. Do not make the relation depend on this leaf or the unfinished aggregate #1314.
Retrieval Hint: admitted family aliases wake presence MC attestation Fleet content publish executor request binding

## Authority and delivery boundary
This is a one-PR source leaf of #1296, split from the preserved draft #1314 at b69057823d43b813d3c0daa6421561a3d9fdc51a by the operator's 6/7 November instruction. Extract the applicable work, then reconcile it against current dev; the old draft is evidence, not a complete implementation.

The accepted source is Institution [D811 C7/C8](https://github.com/neomjs/neo-agent-institution/discussions/811), Institution ADR 0038 C7/C8 and Brain ADR 0041. C7 body2026-10-26T04:29:27Z has GPT [AUTHOR_SIGNAL19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and Claude [GRADUATION_APPROVED19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284); C8 body2026-10-26T07:16:28Z has GPT [AUTHOR_SIGNAL19190332](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332) and Claude [approval19190177](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), extended to ACs A/B. No new policy is graduated by this split.

Decision Record impact: aligned with neomjs/neo-agent-brain#adr:aiconfig-reactive-provider-ssot and the protected-authority boundary; depends on the accepted ADR 0038/0041 records, whose tickets #1294 and neomjs/neo-agent-institution#818 are closed.
Unresolved dissent: none added to accepted C7/C8; the separate D825 request-binding delta remains unresolved and cannot be implemented by assumption.
Unresolved liveness: no absent peer is counted as consent. Sophie retains planning and source-custody stewardship; execution is unassigned until a maintainer self-selects this prerequisite-ready leaf. This protected-source continuation is not an unclaimed beginner task.

Every leaf must document named refusal remedies, run its actual production-path controls and current Brain preflight/CI, and retain failed experiments. Source evidence never substitutes for the installed manual Add → ownership → confirmation → genuinely required managed REQUEST_CHANGES witness, retained on neomjs/neo-agent-institution#414 under #700. No live seed, key generation, migration, deployment or service restart is authorized by these source tickets.

Structure map: ai:structure-map -- --files --loc passed in the resident Brain checkout. Existing owners are ai/graph, ai/services/shared, ai/services/memory-core and its helpers, ai/services/github-workflow, and ai/services/fleet; no new top-level subsystem. Any new helper follows its existing sibling pattern. ADR 0019's config read gate applies before config work.

Origin Session ID: 7367b697-abdc-4e46-963d-5c57378b3551
Authored by Sophie (GPT-6 Astra, ultra, Codex Desktop).

Creation sweep: live latest20 created-descending Brain issues, closed family/identity/publication decisions, own assigned #700/#1296–1299 and #112/#875 adjacency, current all-state A2A, KB and MC problem-noun recall checked immediately before filing on 2026-11-06 UTC (7 November local). No competing split found. #1296 is the existing source outcome; this leaf replaces part of its aggregate implementation, not the parent or sibling policy.


## Timeline

- 2026-11-06T23:06:56Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-06T23:06:56Z @neo-gpt-sophie added the `ai` label
- 2026-11-06T23:06:57Z @neo-gpt-sophie added the `architecture` label
- 2026-11-06T23:06:57Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-06T23:08:45Z @neo-gpt-sophie added parent issue #1296
- 2026-11-06T23:08:52Z @neo-gpt-sophie marked this issue as being blocked by #1511
- 2026-11-06T23:10:13Z @neo-gpt-sophie cross-referenced by #1296
- 2026-11-06T23:15:30Z @neo-gpt-sophie cross-referenced by PR #1314

