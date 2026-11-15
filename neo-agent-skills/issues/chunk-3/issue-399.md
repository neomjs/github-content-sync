---
id: 399
title: Replace review-count polish gates with scoped author consent
state: OPEN
labels:
  - documentation
  - enhancement
  - contributor-experience
  - ai
  - model-experience
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-15T14:09:56Z'
updatedAt: '2026-11-15T16:28:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/399'
author: neo-gpt-sophie
commentsCount: 0
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

For server-side base updates, use the expected-head guard supported by the operation. An accepted asynchronous request is not a completion receipt: verify the resulting head and affected checks before reporting integration done. Tool/API parameter mechanics stay in their owning documentation.

## The Fix / Contract Ledger

| Surface | Authority | Required behaviour | Fallback / edge | Docs / evidence |
| --- | --- | --- | --- | --- |
| §10: content and ticket authority | D#21005 A/B/B1/C/D | Approve a correct first PR; otherwise one complete actionable review and an author-led first meaningful revision unless help is requested sooner. Before each PR’s first content-assistance push, obtain an affirmative author-account reply accepting a named scope. An invitation checkbox is not consent. Subsequent repairs stay within that scope. Record the existing PR-ticket authority used by the revised polish path without replacing the contributor’s assignment. | Withdrawal stops future edits; new behaviour/API/design or substantial replacement needs a new agreement. Missing or ambiguous consent stops content edits. No default takeover or count/label-based inference. | Replace both existing §10 paragraphs. Walk the case table below. |
| Independence and round limits | Step-back 2a | Repairing seat cannot approve; its canonical family is an author family. Preserve the separate reviewer-family demand budget, ordinary Round-2 disposition and guarded re-entry rules. | A repair creates no new demand-round entitlement. Runtime enforcement belongs to the Brain leaf; this change cannot declare it shipped. Human merge and ticket-ID gates remain. | Reconcile the cross-family and intake wording with one policy owner. |
| Mechanical maintenance and integration | §2 | One correct form from named tool/source inputs, no author-intent choice, verified output. Maintain original PR when useful. At most one contributor rebase request for our base churn; use current permission/head-bound route and affected checks. Give `git pull --ff-only` handback and recheck after the next author push. | Stop on local divergence; preserve work for assisted reconciliation. Choosing nonidentical behaviour or discarding meaningful authored work is content even when there is one sensible resolution; changed-line count alone is not the classifier. Never bypass a refusal. | Replace the intake fallback instead of leaving conflicting “exact commands” guidance. |
| Refused prerequisite | §2 / step-back 8 | Public PR receipt names head, operation, limitation and maintainer owner. Owner retains it until an alternate explicitly accepts and actual execution is verified. Existing A2A Task records distinguish Submitted, Working and Completed; completion needs the operation result. | Supported Failed/InputRequired states where allowed; ordinary agent assignees cannot use Rejected as a decline. A sent message or task state does not prove repair, delivery or automatic rotation. Contributor needs no private Task access. | Approve-runs guidance points here; live routed-seat proof remains an activation gate. |
| Dependency date / inactivity | §3 | Named blocked dependency, available author action, at least three working days after the latest legitimate handback adding an action, exact date/time/timezone. Repetition does not reset it. Respect agreed return dates; our review/CI/integration waits do not count. At the date, make a public decision. | Existing claim expiry is unchanged. Ordinary check-ins are separate. A release then recommitment retains the date unless a new agreement changes it; require reopening/revision and reconcile any accepted continuation. No indefinite reset or automatic takeover. | New comments for changed dates, not silent edits. |
| Credit / residual | §4 / step-back 2d | Prefer original PR; when a target already merged, compare the remaining work against current dev and offer a focused residual ticket/retarget before replacement. Preserve retained commit authorship and truthful original-PR merge credit. Support follows preference, risk and demonstrated independence. | Declining help alone does not justify replacement. No new three/five-merge competence threshold or badge count. Unwanted premise may be closed with explanation. | Public guidance parity is separate; no implication another repository already adopted it. |
| Stable receipts | Step-back 3/4 | Versioned hidden offer/refusal markers bind repository, PR and head/operation; GitHub’s comment identity is the durable record key. Consent links its offer. Push receipt preserves the accepted scope, reply author, ID, timestamp and link. | Marker text alone is not authority: validate the expected author/provenance. A known withdrawal prevents acting; later editing/deletion does not erase the recorded accepted scope. Release-marker/dedup wire belongs to E2’s leaf, not a second copy here. | Define the exact offer/refusal marker wire once in a conditional payload, including its field types and examples. Program measurement consumes it; root contract is public. |
| Curated issue entry | E1a | One claim-before-work instruction on the first screen above Context through ticket-create and ticket-triage when labelled later. Preserve the issue’s rationale. | Author’s edit or explicitly authorized guarded update with current body hash; a conflict requires reread. No label-time comment or automated body writer. Baseline/backfill/read belongs to the program leaf. | One compact pointer/sentence in each existing payload, not a new skill. |

## Offer/refusal record wire (new)

One standalone HTML comment outside quoted text or code fences:

`<!-- neo-assistance:v1 {"kind":"offer","head":"<full-head-sha>","operation":"content-repair"} -->`

`<!-- neo-assistance:v1 {"kind":"refusal","head":"<full-head-sha>","operation":"<named-operation>"} -->`

The JSON object has exactly those string fields: `kind` is `offer` or `refusal`, `head` is the complete GitHub head identity being discussed, and `operation` is non-empty. Offer operation is `content-repair`; a refusal names the actual attempted operation. The record key is GitHub’s repository ID, PR number and comment ID; do not invent another local ID. Date and author come from the GitHub envelope. The visible text carries the proposed scope or refusal and named owner. A reply identifies its offer; the push receipt retains the explicit consent snapshot.

Malformed, multiple, unknown-version or foreign-provenance records remain unverified and cannot authorize action or silently become clean measurement data. Copying a marker is not consent. E2 owns its release marker and dedup contract; the measurement leaf consumes both declared contracts. These markers record evidence, not a new action tool or permission channel.

## Acceptance Criteria

- [ ] All six owner surfaces below agree on the ledger; superseded permissions/fallbacks are replaced, not shadowed. Record the source locations changed and the existing ticket-authority exception used by helper edits.
- [ ] A document walk covers: correct first PR; consented residual; absent/declined/withdrawn consent; request for earlier help; return date; maintainer-held CI/integration; semantic conflict; unavailable fork edit; useful residual after original target merged; substantial replacement; returning author in unfamiliar code; and release/recommitment. For each, name next owner, public message, clock, validation and credit from the ledger.
- [ ] The record wire is explicit and consumed by the measurement handoff; unclear/malformed/foreign markers never become consent. Changed dates and withdrawals use new comments; push receipt retains the consent snapshot.
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

No new skill, workflow bot, runtime family/routing implementation, badge rule, claim-limit/expiry change, API-permission bypass or automatic PR merge. Do not copy the Discussion’s chronology or case derivations into loaded rules. Do not treat an auto-filled box as permission or a family representative as proof that every seat can execute.

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

