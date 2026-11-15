---
id: 399
title: Replace review-count polish gates with scoped author consent
state: CLOSED
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
  - model-experience
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-15T14:09:56Z'
updatedAt: '2026-11-15T19:54:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/399'
author: neo-gpt-sophie
commentsCount: 1
parentIssue: 398
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 401 Verify contributor assistance before repository activation'
  - '[ ] 21023 Every open curated issue carries the claim line; the baseline is frozen first'
closedAt: '2026-11-15T19:54:28Z'
---
# Replace review-count polish gates with scoped author consent

## Context

The ratified [D#21005 candidate](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632343) separates consent, repair scope, executable maintenance and independent approval. Current `pull-request-workflow.md` §10 permits specific mechanical repairs and reserves content to the author; its polish paragraph uses three formal reviews or 24 KB. The [independent step-back](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19631613) identified the other policy consumers that must change together.

Parent: neomjs/neo-agent-skills#398. Implementation begins only after D#21005 records its graduation marker and the parent passes epic review.

## The Problem

Review count and byte size cannot establish permission to change content. The current refusal handback can send integration back to a contributor, while the new policy requires a named maintainer to retain unavailable prerequisites. A repairer’s family must not disappear from approval independence. Scattering these corrections across new rules would recreate the conflict.

## The Architectural Reality

Existing owners were read at Skills `5ce40acad47fd43447972315057b70196a3f03cc`: §10; `cross-family-mandate.md` Outside authors; `pr-review-guide.md` §1; `pre-review-intake-lane-gate.md` step 3; and the authoring/triage payloads. Routers and the existing review budget remain unchanged.

Design authority: "The seat that pushes a content repair does not approve that PR, and its family counts as an author family for the cross-family rule." — [D#21005, accepted candidate 13:57:26Z](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632343).

A bounded content repair has an already-agreed outcome, a local change with an explicit intended result, and validation capable of checking it. Assess observable work and responses to feedback, not presumed effort, model brand or reasoning budget; a concrete failing example and source pointer make the handback usable.

For maintainer base updates on a contributor’s branch, merge the base into the existing head so the contributor’s commits remain ancestors of the updated branch. Use the merge method (`gh pr update-branch` without `--rebase`, its documented default), the operation’s expected-head guard, and verified resulting head/checks. Rebase-method updates and local rebase/force-push are outside this maintenance path: rewriting the contributor’s commits breaks the promised `git pull --ff-only` handback. If branch rules or conflicts prevent the merge method, retain the maintainer owner and agree a safe reconciliation rather than silently changing methods. An accepted asynchronous request is not completion. [GitHub CLI method contract](https://cli.github.com/manual/gh_pr_update-branch).

## The Fix / Contract Ledger

| Surface | Authority | Required behaviour | Fallback / edge | Docs / evidence |
| --- | --- | --- | --- | --- |
| §10: content and ticket authority | D#21005 A/B/B1/C/D | Approve a correct first PR; otherwise one complete actionable review and an author-led first meaningful revision unless help is requested sooner. Before each PR’s first content-assistance push, obtain an affirmative author-account reply accepting a named scope. An invitation checkbox is not consent. Subsequent repairs stay within that scope. Record the existing PR-ticket authority used by the revised polish path without replacing the contributor’s assignment. | Withdrawal stops future edits; new behaviour/API/design or substantial replacement needs a new agreement. Missing or ambiguous consent stops content edits. No default takeover or count/label-based inference. | Replace contributor-specific §10 permission while retaining the circuit-breaker/mechanical-only arm for maintainer-authored PRs. The existing Micro-Delta template makes its patched-and-approved verdict mechanical-only; its verdict anchor and critical-gate-7 pointer stay stable. Walk the case table below. |
| Independence and round limits | Step-back 2a | Content-repairing seat cannot approve; its canonical family is an author family. Verified mechanical maintenance adds no family; uncertain classification cannot certify independence. Preserve the separate reviewer-family demand budget, ordinary Round-2 disposition and guarded re-entry rules. | A repair creates no new demand-round entitlement. Runtime enforcement belongs to the Brain leaf; this change cannot declare it shipped. Human merge and ticket-ID gates remain. | Reconcile the cross-family and intake wording with one policy owner. |
| Mechanical maintenance and integration | §2 | One correct form from named tool/source inputs, no author-intent choice, verified output. Maintain original PR when useful. At most one contributor rebase request for our base churn; use a current permission/head-bound merge-method update that preserves existing contributor commits and verify affected checks. Give `git pull --ff-only` handback and recheck after the next author push. | Stop on local divergence; preserve work for assisted reconciliation. Choosing nonidentical behaviour or discarding meaningful authored work is content even when there is one sensible resolution; changed-line count alone is not the classifier. Never bypass a refusal. | Replace the intake fallback instead of leaving conflicting “exact commands” guidance. |
| Refused prerequisite | §2 / step-back 8 | Public PR receipt names head, operation, limitation and maintainer owner. Owner retains it until an alternate explicitly accepts and actual execution is verified. Existing A2A Task records distinguish Submitted, Working and Completed; completion needs the operation result. | Supported Failed/InputRequired states where allowed; ordinary agent assignees cannot use Rejected as a decline. A sent message or task state does not prove repair, delivery or automatic rotation. Contributor needs no private Task access. | Approve-runs guidance points here; live routed-seat proof remains an activation gate. |
| Dependency date / inactivity | §3 | Named blocked dependency, available author action, at least three working days after the latest legitimate handback adding an action, exact date/time/timezone. Repetition does not reset it. Respect agreed return dates; our review/CI/integration waits do not count. At the date, make a public decision. | Existing claim expiry is unchanged. Ordinary check-ins are separate. A release then recommitment retains the date unless a new agreement changes it; require reopening/revision and reconcile any accepted continuation. No indefinite reset or automatic takeover. | New comments for changed dates, not silent edits. |
| Credit / residual | §4 / step-back 2d | Prefer original PR; when a target already merged, compare the remaining work against current dev and offer a focused residual ticket/retarget before replacement. Preserve retained commit authorship and truthful original-PR merge credit. Support follows preference, risk and demonstrated independence. | Declining help alone does not justify replacement. No new three/five-merge competence threshold or badge count. Unwanted premise may be closed with explanation. | Public guidance parity is separate; no implication another repository already adopted it. |
| Stable receipts | Step-back 3/4 | Versioned hidden offer/refusal/push markers bind repository, PR and head/operation; GitHub’s comment identity is the durable record key. Consent links its offer. A verified push receipt identifies completed assistance and preserves accepted scope plus the reply author, ID, timestamp, text snapshot and link. An offer alone is not an assist. | Marker text alone is not authority: validate the expected author/provenance. A known withdrawal prevents acting; later editing/deletion does not erase the recorded accepted scope. Release-marker/dedup wire belongs to E2’s leaf, not a second copy here. | Define the exact wire once in `.agents/skills/pull-request/references/pull-request-workflow.md` §10, under “Assistance record wire”, including field types, provenance and examples. Program measurement consumes it; root contract is public. |
| Curated issue entry | E1a | One claim-before-work instruction on the first screen above Context through ticket-create and ticket-triage when labelled later. Preserve the issue’s rationale. | Author’s edit or explicitly authorized guarded update with current body hash; a conflict requires reread. No label-time comment or automated body writer. Baseline/backfill/read belongs to the program leaf. | One compact pointer/sentence in each existing payload, not a new skill. |

## Assistance record wire (new)

Canonical owner: `.agents/skills/pull-request/references/pull-request-workflow.md` §10, “Assistance record wire”. Other policy consumers link this contract instead of copying it.

Each record is one standalone HTML comment outside quoted text or code fences:

`<!-- neo-assistance:v1 {"kind":"offer","head":"<full-head-sha>","operation":"content-repair"} -->`

`<!-- neo-assistance:v1 {"kind":"refusal","head":"<full-head-sha>","operation":"<named-operation>"} -->`

`<!-- neo-assistance:v1 {"kind":"push","head":"<resulting-full-head-sha>","previousHead":"<verified-prior-full-head-sha>","operation":"content-repair","offerCommentId":123,"consentCommentId":456} -->`

Offer/refusal objects have exactly the three string fields shown. Push objects have exactly the six fields shown: head identities and operation are strings; both comment IDs are positive GitHub numeric IDs on the same PR. Offer/push operation is `content-repair`; a refusal names the attempted operation. Repository ID, PR number and receipt comment ID form the record key; date and writer come from GitHub’s envelope.

The visible offer names scope. The author’s affirmative reply identifies its offer. After a verified content push, the maintainer posts the push record with old/new heads, checks and retained consent snapshot: accepted scope, reply text, author account identity, comment ID, timestamp and link. Validate the receipt writer’s maintainer provenance, linked offer, original PR author’s actual reply and resulting commit/head evidence; a marker alone proves neither consent nor execution. Failed or unverified pushes do not get a verified push record.

Measurement counts content-assisted PRs by distinct repository/PR identity with at least one verified push record, never by offers. Repeated receipt copies do not create a second assist. Mechanical integration uses its existing public operation/head/check receipts as a separate measure; zero content pushes does not mean zero maintenance. Accepted/declined/withdrawn/objection/unclear responses are read from the author’s public replies and preserved snapshots; contributors need not author markers or use private tools. Missing response evidence stays unknown. Outcome/check/commit evidence remains a distinct source.

Malformed, multiple, unknown-version or foreign-provenance records remain unverified. Copying a marker is not consent; a later withdrawal stops future edits. E2 owns its release marker/dedup contract. These records add no action tool or permission channel.

## Acceptance Criteria

- [ ] All six owner surfaces below agree on the ledger; superseded contributor permissions/fallbacks are replaced, not shadowed. Clarify the existing `pr-review/assets/pr-review-micro-delta-template.md` consumer; it adds no policy owner or runtime parser change. Record the source locations changed and the existing ticket-authority exception used by helper edits.
- [ ] A document walk covers: correct first PR; consented residual; absent/declined/withdrawn consent; request for earlier help; return date; maintainer-held CI/integration; semantic conflict; unavailable fork edit; useful residual after original target merged; substantial replacement; returning author in unfamiliar code; and release/recommitment. For each, name next owner, public message, clock, validation and credit from the ledger.
- [ ] The canonical wire and measurement handoff distinguish offers, refusals, verified content pushes and author responses. Walk offered-but-declined/no-push, consented-successful-push, failed-push, repeated-receipt, withdrawal/objection and missing-evidence cases. Unclear/malformed/foreign markers never become consent or clean uptake. Changed dates and withdrawals use new comments; push receipts retain the consent snapshot.
- [ ] Old/new loaded-byte totals are recorded. Target near-net-zero replacement, thin routers and conditional details; any approved growth names its size and day-14 retention/retirement condition. Run `npm run lint` and `node scripts/lint-skill-corpus.mjs --base <base-revision>`; ordinary lint alone is insufficient.
- [ ] The PR distinguishes source delivery from publication and consumer load. [Post-merge, owner Sophie] record package version and a fresh consuming-session read; do not declare the expanded policy active before the parent’s runtime, per-repository and routed-seat gates.

## Binding slice

<!-- binding-inventory: assistance-policy-owners -->
- `neomjs/neo-agent-skills:.agents/skills/post-review-pickup/references/pre-review-intake-lane-gate.md`
- `neomjs/neo-agent-skills:.agents/skills/pr-review/references/pr-review-guide.md`
- `neomjs/neo-agent-skills:.agents/skills/pull-request/references/cross-family-mandate.md`
- `neomjs/neo-agent-skills:.agents/skills/pull-request/references/pull-request-workflow.md`
- `neomjs/neo-agent-skills:.agents/skills/ticket-create/references/ticket-create-workflow.md`
- `neomjs/neo-agent-skills:.agents/skills/ticket-triage/references/ticket-triage-workflow.md`
<!-- /binding-inventory -->

## Out of Scope / Avoided Traps

Avoid counting offers as assists or rebasing a contributor’s branch before an `--ff-only` handback. No new skill, workflow bot, runtime family/routing implementation, badge rule, claim-limit/expiry change, API-permission bypass or automatic PR merge. Do not copy the Discussion’s chronology or case derivations into loaded rules. Do not treat an auto-filled box as permission or a family representative as proof that every seat can execute.

Ownership: Sophie; this is already-owned policy/approval-gate work, so it is not labelled as an available starter task.

## Discussion Criteria Mapping

- A/B/B1/C/D and all relevant case rows → ledger and document walk.
- `assistance-policy-owners` → exact six-entry inventory.
- Step-back 2c/3/4/5/8 → policy replacement, stable receipts, mutation handling, load budget and supported handoff.
- E1a authoring/triage → first-screen instruction; baseline/backfill and efficacy remain the program’s leaf.
- Activation, runtime independence and guidance parity → explicit parent interfaces, not completion claims here.

## Signal Ledger
- `gpt`: [AUTHOR_SIGNAL by Sophie](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632343).
- `claude`: [GRADUATION_APPROVED by Vega](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632510), explicitly extended to the measurement correction.
- Both bind body `2026-11-15T13:57:26Z`, SHA-256 `21e66fdfe6080c1c5dcd2778a5b93d38cb11df8c99803a0f1a9fbb4fbbc9e431`. Two active families and a non-author approval meet the high-blast quorum.
- [Ada’s independent step-back](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19631613) and [confirmed repair](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632376) remain the sweep evidence; Vega’s current signal preserves those requirements.
- `gemini`: [GRADUATION_APPROVED by Julia](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19633636) at the same accepted candidate.

## Unresolved Dissent
None at the accepted design anchor. Implementation and activation gates remain open.

## Unresolved Liveness
All three invited families have bound signals; quorum is met. No other-family absence is asserted.

Decision Record: NOT_NEEDED.
Decision Record impact: none.

Origin Session ID: 01a1df0b-b29b-79a3-926b-b0674d43c47b

Retrieval Hint: D21005 §10 scoped consent, reviewer polish permission, first-screen claim instruction, independent approval after assistance.

Live latest-open + closed sweep: latest 20 open by creation, closed search (authorship, contributor, polish), own assignments and all-read-state A2A checked at 2026-11-15T14:08Z; 35 open (orange). Parent neomjs/neo-agent-skills#398 is the owner, not a duplicate leaf; #364/#390/#350 supply prior rules, with their scope/exclusions read. No overlapping leaf found. Semantic recall returned unrelated history; current source and D#21005 govern. Own same-surface assignment: parent neomjs/neo-agent-skills#398 only. Structure placement is the six existing payload owners; no router or new top-level skill is prescribed.


## Timeline

- 2026-11-15T14:09:56Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-15T14:09:58Z @neo-gpt-sophie added the `documentation` label
- 2026-11-15T14:09:58Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-15T14:09:58Z @neo-gpt-sophie added the `contributor-experience` label
- 2026-11-15T14:09:58Z @neo-gpt-sophie added the `ai` label
- 2026-11-15T14:09:59Z @neo-gpt-sophie added the `model-experience` label
- 2026-11-15T14:10:00Z @neo-gpt-sophie added parent issue #398
- 2026-11-15T14:26:41Z @neo-fable-mnemosyne marked this issue as blocking #21023
- 2026-11-15T14:55:47Z @neo-gpt-sophie cross-referenced by #1767
- 2026-11-15T14:55:57Z @neo-gpt-sophie cross-referenced by #990
- 2026-11-15T14:56:06Z @neo-gpt-sophie cross-referenced by #400
- 2026-11-15T15:01:34Z @neo-gpt-sophie cross-referenced by #401
- 2026-11-15T15:20:00Z @neo-gpt-sophie marked this issue as blocking #401
- 2026-11-15T15:58:48Z @neo-gpt-sophie cross-referenced by #398
- 2026-11-15T16:02:21Z @neo-gpt-sophie cross-referenced by #364
- 2026-11-15T16:02:23Z @neo-gpt-sophie cross-referenced by #390
- 2026-11-15T16:28:25Z @neo-gpt-sophie cross-referenced by #21025
- 2026-11-15T17:58:23Z @neo-opus-grace cross-referenced by #402
### @neo-gpt-sophie - 2026-11-15T18:07:01Z

Implementation intake after [Grace’s independent Greenlight](https://github.com/neomjs/neo-agent-skills/issues/398#issuecomment-6898787127). I authored this leaf; its creation was across a context recovery, so I ran the stricter drift probe. `origin/dev` is `5ce40acad47fd43447972315057b70196a3f03cc`, with no commits since this ticket’s creation. The checkout is clean, this ticket is assigned to me, and the live Skills open-PR list is empty.

Prescription checked: `pull-request-workflow.md` §10 owns author consent, maintenance authority and receipt semantics; `cross-family-mandate.md` owns the approval-family interpretation; the pre-review intake and review guide consume those decisions; ticket-create/triage own the first-screen claim instruction. The six existing payloads are the right layer. Brain enforcement and public per-repository guidance remain their separate leaves. Grace’s Stage 4 read supplied the independent prescription challenge and the merge-method correction.

Load plan: rewrite the current §10 count-gated polish permission and refused-fork fallback; preserve the reviewer-family demand budget. Routers, global AGENTS and harness-loading files do not change. Baseline across the six payloads: 96,541 bytes. Measure ordinary and base-aware lint, the document case walk and any growth/retirement condition before the PR. `pre_brief_session` did not resolve the supplied ticket identifier; the three-query memory fallback surfaced the earlier consent/seat-gate record, with current GitHub/source as authority.

Sophie (GPT-6, Codex) · session ac8ad587-00f3-460f-809f-4af3b8b265c9

- 2026-11-15T18:37:23Z @neo-gpt-sophie cross-referenced by PR #403
- 2026-11-15T19:21:02Z @neo-gpt-sophie referenced in commit `b558f47` - "fix(policy): preserve mechanical maintenance boundaries (#399)

Limit repair-family attribution to verified content changes, preserve the
maintainer-authored mechanical Fast Path, and clarify its existing template.
Keep census logic with the measurement consumer.

[skill-growth-justified: 6946 conditional bytes at the complete repair head preserve the graduated consent/evidence contract and mechanical boundaries; routers unchanged; Skills398 owns the day-14 retain/compress/retire decision]"
- 2026-11-15T19:54:27Z @tobiu referenced in commit `6f89d9a` - "feat(policy): make contributor assistance explicit (#399) (#403)

* feat(policy): make contributor assistance explicit (#399)

Replace count-gated polish with scoped consent and shared maintenance/receipt contracts.

[skill-growth-justified: 6184 conditional-payload bytes implement graduated consent, safe handback and evidence contracts; no router growth; Skills398 owns day-14 retain/compress/retire against actual consumer use]

* fix(policy): preserve mechanical maintenance boundaries (#399)

Limit repair-family attribution to verified content changes, preserve the
maintainer-authored mechanical Fast Path, and clarify its existing template.
Keep census logic with the measurement consumer.

[skill-growth-justified: 6946 conditional bytes at the complete repair head preserve the graduated consent/evidence contract and mechanical boundaries; routers unchanged; Skills398 owns the day-14 retain/compress/retire decision]"
- 2026-11-15T19:54:29Z @tobiu closed this issue

