---
id: 400
title: Expose scoped help in the Skills contributor entry
state: OPEN
labels:
  - documentation
  - enhancement
  - help wanted
  - contributor-experience
  - ai
assignees: []
createdAt: '2026-11-15T14:56:05Z'
updatedAt: '2026-11-15T23:32:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/400'
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
---
# Expose scoped help in the Skills contributor entry

Please comment to claim before starting; check current assignees and linked PRs.

## Context

D#21005 selects Engine-first assistance guidance with explicit parity before the policy activates in sibling repositories. Engine’s public contract is recorded in neomjs/neo#21025; the maintainer rule is neomjs/neo-agent-skills#399. This is the Skills entry-surface leaf under neomjs/neo-agent-skills#398.

## The Problem / Architectural Reality

The existing public entry is `README.md`. Preserve its first 40 source lines: #359 is measuring the opening selected by #356. Put assistance guidance in a contribution section below that opening; this is not an identity or opening rewrite. This repository has no repository-local root PR template in the verified dev tree, so the choice must also work for authors opening a PR through the CLI or an agent. A guide in Engine alone does not establish what a Skills contributor can request.

Design authority: "Guidance and the optional invitation start in Engine. Brain, Institution and Skills carry explicit template/guide parity leaves before the expanded policy activates there." — [D#21005, accepted candidate 13:57:26Z](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632343).

## The Fix / Contract Ledger

| Surface | Required behaviour | Fallback / edge | Evidence |
| --- | --- | --- | --- |
| `README.md` contribution entry | Explain that a correct first PR can be approved, otherwise feedback is actionable and the author can request help. An optional invitation in the PR description welcomes an offer; each PR’s first content-assistance push requires an affirmative reply accepting a specific scope. Author can decline or withdraw; later changes stay within that agreement. | No checked/default field or edit-access flag is consent. No label or merge-count competence test; no required private tool or internal ceremony for the contributor. | Independent entry-only read explains the choice and next step. |
| Maintenance / time / credit | Mechanical base churn is maintainer-owned where authorized; at most one author rebase request for our moving base. Maintainer base updates preserve existing contributor commits by merging the base; handback is `git pull --ff-only`, with preserved local work if divergent. Our CI/review/integration waits do not count as author inactivity. Keep original PR credit and offer retargeting for a useful residual after the original issue merged. | Semantic conflicts/new behaviour need agreement. Dependency dates name an available author action and at least three working days from the latest legitimate action-adding handback, with date/timezone; unchanged repeat actions do not reset them. Ordinary check-ins stay separate; existing claims/standing are unchanged. A declined offer alone does not justify replacement. | Brief examples of consent, our wait, conflict and residual. |
| Repository activation | Public wording agrees with the shared policy and this repository’s actual claim/standing conventions. Coordinate a dated activation after the parent’s current-seat, runtime and consumer gates. | Do not infer that Engine’s source change, workflow or capability applies here. Preserve existing disclosure/close-target conventions and any inherited template behaviour. | Source revision plus repository-specific activation handoff. |

## Acceptance Criteria

- [ ] Implement every Required behaviour and Fallback / edge in the Contract Ledger. The first 40 README lines stay unchanged while #359’s reader study is open; the added contribution section sits below them.
- [ ] The existing contribution entry gives the complete choice in ordinary language and a usable description/comment invitation. It clearly requires scoped affirmative consent before content edits and preserves withdrawal.
- [ ] The reader can identify the next owner, safe local-branch handback and truthful credit, including an already-merged original close target.
- [ ] Claim limits/expiry, badge definitions, existing disclosure requirements and dev/close-target rules are preserved. Do not copy Engine-specific standings into this repository without their separate workflow delivery.
- [ ] Links and the repository’s applicable documentation checks pass; no mirror unit test for prose is added.
- [ ] The PR hands the exact source revision to the activation owner. This leaf is source parity, not a claim of loaded policy, working permissions, automatic rerouting or installed acceptance.

## Out of Scope / Avoided Traps

No new default template, workflow implementation, permission change, issue-body backfill, review predicate or new skill. Keep record-wire details with the shared maintainer policy rather than exposing them as contributor obligations.

unowned-rationale: Ready for contributor pickup. D#21005 is graduated and the shared policy in #399 shipped through [PR #403](https://github.com/neomjs/neo-agent-skills/pull/403), published as `neo-agent-skills@1.9.10` ([source/publication receipt](https://github.com/neomjs/neo-agent-skills/issues/401#issuecomment-6900766523)). This leaf stays unassigned for a contributor; Sophie provides the parent’s planning and review handoff. Source readiness does not activate expanded assistance: #401 retains the consumer, runtime and operation gates.

## Discussion Criteria Mapping

Step-back 2d and the guidance-parity row → Skills’s explicit source leaf. A/B/B1/C/D and the relevant case matrix → the choice/time/credit contract above. Activation remains the parent’s separately receipted gate. This leaf has no source-marked enumerable slice; its coverage is judged.

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

Retrieval Hint: D21005 Skills contributor guidance parity, scoped help invitation and consent, original PR credit.

Live latest-open + closed sweep: latest 20 created-descending Skills issues and closed contributor/guidance/assistance search read immediately before filing; 36 open (orange). Existing neomjs/neo-agent-skills#398 owns this parity. Own assignments and current source were read at 14:51Z; none owns this public-help entry. Closed guide-creation/claim-window scopes and relevant exclusions were read. #356/#359 protect the opening; this leaf explicitly leaves it unchanged. A2A all-read-state scope check found no competing guidance claim. Semantic recall supplied no competing same-surface decision. No runtime module or new directory is prescribed.


## Timeline

- 2026-11-15T14:56:06Z @neo-gpt-sophie added the `documentation` label
- 2026-11-15T14:56:06Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-15T14:56:07Z @neo-gpt-sophie added the `contributor-experience` label
- 2026-11-15T14:56:07Z @neo-gpt-sophie added the `ai` label
- 2026-11-15T14:56:08Z @neo-gpt-sophie added parent issue #398
- 2026-11-15T14:57:56Z @github-actions added the `needs-design-authority` label
- 2026-11-15T15:01:34Z @neo-gpt-sophie cross-referenced by #401
- 2026-11-15T15:13:39Z @github-actions removed the `needs-design-authority` label
- 2026-11-15T15:20:02Z @neo-gpt-sophie marked this issue as blocking #401
- 2026-11-15T15:58:48Z @neo-gpt-sophie cross-referenced by #398
- 2026-11-15T21:00:56Z @neo-gpt-sophie added the `help wanted` label
### @neo-gpt-sophie - 2026-11-15T21:00:58Z

This documentation task is ready for pickup: the shared policy prerequisite has shipped in [#403](https://github.com/neomjs/neo-agent-skills/pull/403).

The deliverable is a contribution section **below README line 40** that lets a contributor understand the help invitation, scoped agreement, withdrawal and safe branch handback without learning our internal record format. Use the Contract Ledger above as the coverage checklist and the [merged maintainer policy](https://github.com/neomjs/neo-agent-skills/blob/6f89d9a33643f156f562eb72651698d31ac5a8c1/.agents/skills/pull-request/references/pull-request-workflow.md) as the reference. Keep the first 40 lines unchanged.

If you would like to take it, comment here so we can confirm ownership before work starts. I can clarify the policy and arrange the independent entry-only read. This issue remains unassigned; the `ai` label records its authoring provenance, not a reservation.


