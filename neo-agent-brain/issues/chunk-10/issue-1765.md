---
id: 1765
title: Count maintainer repair authors in outside PR approval eligibility
state: OPEN
labels:
  - enhancement
  - contributor-experience
  - ai
  - testing
  - agent-os
assignees: []
createdAt: '2026-11-15T14:33:30Z'
updatedAt: '2026-11-15T21:16:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1765'
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
blockedBy:
  - '[ ] 1512 Use admitted family history for review decisions'
blocking:
  - '[ ] 401 Verify contributor assistance before repository activation'
---
# Count maintainer repair authors in outside PR approval eligibility

## Context

[Ada’s independent step-back](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19631613) found an approval-independence gap in the new assistance path. D#21005’s ratified rule is: “The seat that pushes a content repair does not approve that PR, and its family counts as an author family for the cross-family rule.”

Parent outcome: neomjs/neo-agent-skills#398. Do not begin implementation before the source graduation marker and parent review.

## The Problem / Architectural Reality

At Brain `d6e65d750144c87acf6509f9f7d7065dea46134d`, `resolveCrossFamilyVerdict` selects an outside PR’s author families from `pr.disclosedFamilies ?? disclosedFamiliesOf(pr)` (lines 563–610). A maintainer repair commit need not carry a model-disclosure trailer, so the repairing family can remain invisible. `openWorkHolder` also consumes the outside-family set when deciding which approval counts.

Design authority: "The seat that pushes a content repair does not approve that PR, and its family counts as an author family for the cross-family rule." — [D#21005, accepted candidate 13:57:26Z](https://github.com/neomjs/neo/discussions/21005#discussioncomment-19632343).

The admitted authority and existing temporal semantics remain owned by #700/#1296 and their consumer leaves. This change adds commit identities to those existing readers; it introduces no static, harness, display or commit-date fallback.

## Source readiness — intake 2026-11-15

Blocked by #1512, the one-PR delivery leaf for admitted family history in review decisions. Its open #1511 dependency remains upstream; this leaf does not duplicate their protected source/client or temporal classification work.

The current [readiness caller at `c2b7859a`](https://github.com/neomjs/neo-agent-brain/blob/c2b7859a42b280b1126d040832623c61247703d4/ai/services/github-workflow/PullRequestService.mjs#L998-L1012) still passes the legacy author/reviewer family resolution into the verdict. #1512 explicitly owns replacing those runtime inputs with the qualified admitted source, including immutable forge actors, complete temporal author-family union and removal of static/flat fallback. This ticket's existing no-fallback contract therefore cannot be implemented as though that source were already delivered.

Prescription checked: `agentFamilyResolution.mjs#resolveCrossFamilyVerdict` and `openWorkHolder.mjs#holderOf` own the repair-family exclusion and its shared approval-consumer facts; #1512 owns their prerequisite review authority. After #1512 lands, re-read the resulting interfaces and current #1766/#1768 surface before claiming. #1513's separate alias/publication work and the aggregate #1296 epic are not blanket blockers for this leaf.

The [independent parent review is green](https://github.com/neomjs/neo-agent-skills/issues/398#issuecomment-6898787127). Draft #1314 is preserved extraction evidence under the [operator-authorized source split](https://github.com/neomjs/neo-agent-brain/issues/1296#issuecomment-6431673408), not a prerequisite merge candidate. No policy or acceptance criterion changes in this intake correction.

## The Fix / Contract Ledger

| Surface | Proposed behaviour | Fallback / edge | Evidence |
| --- | --- | --- | --- |
| Outside PR author-family resolution | Union original-author/disclosed families with canonical families of verified maintainer content-repair authors in the current PR commit set. Classify from the actual diff and public operation evidence under the shared policy: a base-only merge without semantic conflict choices, or verified mechanical hygiene/metadata, adds no author family. A merge containing semantic resolution remains content; a title or marker alone is not classification. Use the existing admitted identity/family resolver inputs, including canonical/former-login handling; never infer family from a handle or model name guess. | GitHub commit identity, not arbitrary message text, establishes the contributing seat. Preserve existing unknown-family policy. Missing/truncated identity, commit or classification evidence cannot silently yield a positive independence verdict. | Pure predicate matrix plus complete-read controls. |
| Reviewer eligibility and readiness | The content-repairing seat cannot approve the PR; an approval from any author family does not satisfy cross-family coverage. A qualifying other-family approval can satisfy the existing rule when evidence is complete. | Preserve current-head validation and review-budget rules. A new head re-reads contributing families; old approval cannot silently certify a newly authored repair. | Actual merge-readiness/predicate consumer tests. |
| Fleet projection / routing readers | Feed the same derived author-family facts to readers that choose/count outside review, so the cockpit, holder and wake path do not disagree with readiness. | Preserve partial/unavailable states and existing fallback policy; do not hide missing data behind an empty family list. | End-to-end source-consumer fixture through the existing projection, plus post-merge live receipt. |

## Acceptance Criteria

- [ ] An outside PR with no model disclosure plus a maintainer content repair excludes that seat and family from approval coverage. Negative controls: base-only merge and verified mechanical maintenance add no family; semantic conflict resolution does, even in a merge commit.
- [ ] Same-family approval fails coverage; eligible different-family approval passes only against complete current-head evidence. Test multiple contributing families and the original disclosed family union.
- [ ] Missing/unknown identity, truncated/unread commit history, absent data and a newly changed head retain explicit uncertainty/failure rather than becoming “unnamed”.
- [ ] Known aliases use the existing identity contract; no copied roster or new direct runtime identity-root reader is introduced.
- [ ] All actual consumers are identified and updated/tested, including merge readiness and Fleet outside-review eligibility in `ai/services/fleet/openWorkHolder.mjs#holderOf`. At Brain `ed48a7052b`, its `named`/`counts`/`approved` block counts every approval when the outside PR has no disclosure; replace that counting copy with the same admitted derived author-family/eligibility facts as the verdict. Test an undisclosed outside PR with a repairing seat’s on-head approval through the holder as well as the verdict, including unavailable/truncated evidence. A helper-only green test is insufficient.
- [ ] Existing family-budget, exception and current-head semantics remain intact except the newly counted content-repair families.
- [ ] [Post-merge, maintainer owner] record one live outside-PR repair/approval receipt against the deployed revision. Until this gate is witnessed, the expanded assistance path is not activated.

## Binding slice
<!-- binding-inventory: assistance-runtime-owners -->
- `neomjs/neo-agent-brain:ai/services/graph/agentFamilyResolution.mjs#resolveCrossFamilyVerdict`
<!-- /binding-inventory -->

## Out of Scope / Avoided Traps

No permission setting, model-family roster/history migration, review-budget expansion, new approval wrapper or policy bypass. Do not “fix” this only by asking helpers to remember a disclosure line; the actual commit identity is the missing input.

unowned-rationale: Maintainer gate work after graduation; it changes approval eligibility and needs installed verification. It is not advertised as an unblocked starter task. Sophie owns the parent’s acceptance/handoff.

## Discussion Criteria Mapping
- Step-back 2a and §1 → unioned author families, repairing-seat exclusion, complete-evidence matrix.
- `assistance-runtime-owners` → this exact predicate owner; the holder’s approval-counting consumer belongs here; outside integration routing/wakes belong to neomjs/neo-agent-brain#1766. Coordinate the shared `holderOf` surface at intake with #1766 and #1768 rather than broadening either leaf.
- Activation and source-versus-live distinction → deployed receipt, not source CI alone.

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

Retrieval Hint: outside PR co-completion self approval, maintainer commit family missing, resolveCrossFamilyVerdict contributor repair.

Live latest-open + closed sweep: latest 20 Brain issues by creation, closed family/approval search, own assignments and all-read-state A2A checked in this filing window; 160 open (red). The new leaf serves existing neomjs/neo-agent-skills#398. Closed #1702’s scope/exclusions and own #700/#1296 bodies were read: disclosure parsing and admitted family authority remain their owners; this leaf adds contributing commit identities to the existing predicate. No equivalent repair-author consumer leaf found. Current owner source verified at d6e65d75; structure map and exact path reads support existing placement. No new module/directory is prescribed. MC recall surfaced #700 history, not a competing repair-author decision.


## Timeline

- 2026-11-15T14:33:32Z @neo-gpt-sophie added the `enhancement` label
- 2026-11-15T14:33:32Z @neo-gpt-sophie added the `contributor-experience` label
- 2026-11-15T14:33:32Z @neo-gpt-sophie added the `ai` label
- 2026-11-15T14:33:32Z @neo-gpt-sophie added the `testing` label
- 2026-11-15T14:33:32Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-15T14:33:35Z @neo-gpt-sophie added parent issue #398
- 2026-11-15T15:20:10Z @neo-gpt-sophie marked this issue as blocking #401
- 2026-11-15T15:58:48Z @neo-gpt-sophie cross-referenced by #398
- 2026-11-15T16:02:24Z @neo-gpt-sophie cross-referenced by #1702
- 2026-11-15T16:28:20Z @neo-gpt-sophie cross-referenced by #401
- 2026-11-15T18:50:09Z @neo-opus-grace cross-referenced by PR #403
- 2026-11-15T21:16:12Z @neo-gpt-sophie marked this issue as being blocked by #1512
- 2026-11-15T21:28:40Z @neo-gpt-sophie cross-referenced by #1509

