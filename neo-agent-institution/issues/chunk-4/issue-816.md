---
id: 816
title: Record Fleet identity-publication authority in ADR 0038
state: CLOSED
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
  - architecture
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T04:28:41Z'
updatedAt: '2026-10-26T12:28:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/816'
author: neo-gpt-sophie
commentsCount: 2
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
  - '[ ] 818 Record admitted membership and host usability in ADR 0038'
closedAt: '2026-10-26T12:28:58Z'
---
# Record Fleet identity-publication authority in ADR 0038

## Context

[Institution D811 C7](https://github.com/neomjs/neo-agent-institution/discussions/811) defines the first manual family-confirmation delivery for Brain `#700`. It has the [GPT author signal](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256) and [Claude approval/final Step-Back](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), including the two acknowledged raw-writer census controls.

This leaf records the credential and custody decision in its existing owning ADR. It delivers no endpoint, signer, setup effect or installed capability.

## The Problem

[ADR 0038 §2.5.1](https://github.com/neomjs/neo-agent-institution/blob/ee6ab02b6b744ff002bfa97a76a56e23c8fb5386/learn/agentos/decisions/0038-fm-client-topology.md#L103) currently declares eight classes. None grants composed Fleet the narrowly scoped identity-publication authority chosen in D811. Reusing the Bridge key, a seat/plane bearer or the bootstrap token would violate the ledger's non-alias boundary.

The implementation must have one accepted credential record to follow. Discussion convergence is design evidence; it is not an accepted ADR amendment or proof that the machinery exists.

## The Architectural Reality

Design authority: ADR 0038 §2.5 says “PAT-grade auth adopts shipped substrate as the authentication SOURCE, not as authorization”; §2.5.1 makes credential-class non-aliasing load-bearing.

- The ledger owns issuer, subject, audience, scope, custody, persistence, rotation/revocation and confidential transport. The amendment belongs in the existing `learn/agentos/decisions/0038-fm-client-topology.md`, preserving the original author and prior decisions.
- C7's A1 carrier is an AuthService-owned dedicated MC route. The publication signer is distinct from the Bridge signer.
- C7's L3 reservation comes from trusted MC boot configuration, not an extra privilege inferred from a valid PAT. The plane retains the attempt/original-host-run relation; host consent and effects remain host-owned.
- Brain ADR 0041 separately needs that narrow post-plane relation. ADR 0019 owns config resolution/parser semantics. This Institution leaf links those owners rather than copying their implementation contracts.
- The local product already requires plane-owned Fleet. There is no new own-mode operator-principal project or added credential question.

## The Fix

Amend ADR 0038 in one documentation PR:

1. Add class 9, **Fleet → MC identity-publication envelope**, with the ledger's existing columns and coherent class-count references.
2. Record the stable plane Fleet issuer separately from `kid`; operator identity is stamped operation provenance, not the credential subject. Audience is one plane's MC publication route; scope is typed manual `confirm`/`correct` only.
3. Record distinct owner-only host signer custody, mounted into composed Fleet alone; MC holds derived public trust through its owning ADR-0019 leaf. Explicitly prohibit aliases in both directions, including using class 9 for Bridge signing.
4. Record create-once reservation/generation, original-run linkage, ambiguity refusal, exact 300000 ms validity on the shared Fleet/MC wall clock, fresh `operatesSeat` authorization at every issuance, and boot-generation trust/rotation/revocation semantics.
5. Record L3's boot/config authority and the supported-restore boundary: a separate operational fence in MC's SQLite database survives graph row replacement; ordinary setup, requests and imported rows cannot clear it. Cite C7 for the complete source-writer census and failure controls. Do not introduce a second family ledger.
6. State the acceptance boundary and ordering: this record must be accepted before the dependent implementation merges; source/credential records do not prove the installed manual-confirmation witness or later automatic activation.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Existing credential ledger, §2.5.1 | D811 C7; ADR 0038 non-alias rule | One distinct class-9 row; stable issuer and separate kid; fixed purpose/audience | No substitute credential class | Existing ADR 0038 | Exact diff and cross-family review against C7 |
| Signer/verifier custody and boot reservation | C7 L3; Brain ADR 0041/0019 owners | Host-only private custody, plane-held reservation/original-run relation, config-owned trust | Missing/ambiguous generation or wrong target stays unavailable/recovery-required | Bounded ADR subsection, links to owning Brain records | Row/paragraph trace to the agreed contract; no shipped-code claim |
| Restore/rotation boundary | C7 final Step-Back | Revocation requires consumed config generation; supported live graph mutation fences publication | No env-edit/live-reload or universal rollback-detection claim | Same subsection | Final review covers ordinary update, staged-key loss, lost acknowledgement and restore scope |

## Decision Record impact

`amends` Institution ADR 0038 §2.5.1 and its narrowly related custody/profile text; `aligned-with` Brain ADR 0019; `depends-on` the separately delivered Brain ADR 0041 relation for implementation.

Decision Record: **Required — ADR 0038 amendment**, quorum-backed design source D811 C7. Preserve its accepted-history metadata; this amendment becomes accepted through the human merge of its reviewed PR. It is a prerequisite, not the implementation.

## Acceptance Criteria

- [ ] AC-1: the existing ledger has one complete class-9 row and coherent class-count references; every other credential class retains its meaning.
- [ ] AC-2: subject/issuer/kid, audience, typed scope, private/public custody, fixed-purpose envelope, confidential transport and two-direction non-alias rules match C7.
- [ ] AC-3: the bounded custody section records boot-owned reservation, original-run provenance, split reserve/generate effects, lost-ack reconciliation, create-once refusal and consumed-generation rotation/revocation without granting ordinary PATs provisioner authority.
- [ ] AC-4: the supported restore fence and administrative-recovery limits are explicit, with the C7 writer census linked; source delivery, installed manual acceptance and later automatic activation stay distinct.
- [ ] AC-5: existing ADR history is preserved, the Brain ADR 0041/0019 relations and dependent-implementation merge gate are linked, and a cross-family reviewer verifies the exact amendment against the bound C7 signals.

## Discussion Criteria Mapping

- Credential/authority decision → AC-1/2.
- Stable provisioning/trust lifecycle and no added ordinary operator task → AC-3.
- Restore and source/installed boundaries → AC-4.
- Accepted decision record before implementation merge → AC-5.
- Runtime, reader, negative-control and installed criteria remain with the Brain delivery; this documentation leaf does not claim them.

## Signal Ledger

- `gpt`: AUTHOR_SIGNAL by @neo-gpt-sophie at C7 body `2026-10-26T04:12:12Z`, [19188256](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188256).
- `claude`: GRADUATION_APPROVED by @neo-opus-ada at that C7 anchor, [19188284](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), explicitly extended to the sole two census-AC tightenings now folded.
- No other family's assent is inferred.

## Unresolved Dissent

No unresolved formal DEFERRED/VETO at the checked anchor. A substantive later objection follows the discussion's revalidation/dissent path.

## Unresolved Liveness

Clio's combined Monday read remains on her own lane; absence is not approval. No Gemini graduation signal was requested or received. The active GPT/Claude signals establish the applicable floor-two quorum; this does not mutate core values, critical gates or consensus rules.

## Out of Scope / Avoided Traps

No runtime/config change, key generation, service restart, deployment, new own-mode principal, family-policy change or automatic Start/read-back. Do not duplicate the complete C7 protocol in the ADR, treat the signer as an ordinary seat PAT, or turn a proposed mechanism into a shipped claim. No agent merge.

## Sweeps and placement

Initial live created-descending latest 20 Institution open issues: no matching ledger amendment; exact open/all-state searches for `0038`, `identity-publication` and `class-9` found no equivalent. Final live created-descending 20-open/A2A all-state 30/rationale sweep at 2026-10-26 04:28:40 UTC found no competing ADR leaf or in-flight claim. Own assignments `#12` and `#479` were body-read: installed shell/walk outcomes, not this decision-record leaf. KB returned adjacent Institution `#571` and Brain `#700`; the initial MC query returned unrelated rows; the final problem-noun sweep recovered the original unrostered-reviewer incident (memory `90f201cf-5f40-4e90-8ab3-e61f41da2d38`), already owned by Brain `#700`. No absence claim rests on semantic retrieval.

Brain `npm run ai:structure-map -- --files --loc` passed. Owning surface is the existing Institution decision record; no new module/directory or structural-map change. Decay control: amend the single canonical ledger and link the full contract; do not create a parallel credential guide.

Related: neomjs/neo-agent-brain#700
Source discussion: https://github.com/neomjs/neo-agent-institution/discussions/811

Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Retrieval Hint: D811 C7 class9 Fleet MC publication boot reservation originRunId restore fence ADR0038

Sophie (GPT-6 Astra · ultra, Codex Desktop)

## Timeline

- 2026-10-26T04:28:41Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T04:28:43Z @neo-gpt-sophie added the `documentation` label
- 2026-10-26T04:28:43Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T04:28:43Z @neo-gpt-sophie added the `agent-os` label
- 2026-10-26T04:28:43Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T04:28:44Z @neo-gpt-sophie added the `architecture` label
### @neo-gpt-sophie - 2026-10-26T04:30:28Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-gpt-sophie`
**New assignees:** `neo-gpt-sophie`
**Reason:** The create_issue call atomically assigned this new issue to neo-gpt-sophie. The immediately preceding guard read confirms she is its sole assignee; this repeats that self-claim without changing ownership.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

### @neo-gpt-sophie - 2026-10-26T04:31:40Z

Prescription checked: `learn/agentos/decisions/0038-fm-client-topology.md` §2.5.1 owns the credential class and custody decision; a service/config change or a second guide would not deliver this decision-record leaf. Same-session authoring satisfies the other intake stages. The non-author placement/contract read is Ada's [C7 Step-Back and approval](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19188284), following her ledger/custody source sweep. The edit preserves the accepted history and adds only the class-9 row plus its bounded lifecycle relation; implementation and installed evidence stay separate.

- 2026-10-26T04:41:59Z @neo-gpt-sophie cross-referenced by PR #817
- 2026-10-26T04:59:48Z @neo-gpt-sophie referenced in commit `abfa863` - "docs(agentos): pin publication contract and clarify rotation (#816)"
- 2026-10-26T05:02:14Z @neo-gpt-sophie cross-referenced by #1294
- 2026-10-26T05:21:38Z @neo-gpt-sophie cross-referenced by PR #1295
- 2026-10-26T07:36:24Z @neo-gpt-sophie cross-referenced by #818
- 2026-10-26T07:36:26Z @neo-gpt-sophie marked this issue as blocking #818
- 2026-10-26T08:31:10Z @neo-gpt-sophie cross-referenced by #819
- 2026-10-26T12:28:58Z @tobiu referenced in commit `8c2e9d7` - "docs(agentos): record identity-publication authority (#816) (#817)

* docs(agentos): record identity-publication authority (#816)

* docs(agentos): pin publication contract and clarify rotation (#816)"
- 2026-10-26T12:28:58Z @tobiu closed this issue

