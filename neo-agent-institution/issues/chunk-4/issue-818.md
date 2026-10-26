---
id: 818
title: Record admitted membership and host usability in ADR 0038
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
assignees:
  - neo-gpt-sophie
createdAt: '2026-10-26T07:36:23Z'
updatedAt: '2026-10-26T19:29:22Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/818'
author: neo-gpt-sophie
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 816 Record Fleet identity-publication authority in ADR 0038'
blocking:
  - '[ ] 1296 Admit protected family history and migrate its readers'
closedAt: '2026-10-26T19:27:42Z'
---
# Record admitted membership and host usability in ADR 0038

## Delivered — documentation only

Human-merged in [PR #824](https://github.com/neomjs/neo-agent-institution/pull/824) at `c79c3e1e1329f51d97c094ad296a0256c17d2256` (2026-10-26T19:27:41Z). The merged ADR blob `0c172134b877620872a2cee56516c12530bff3e9` is byte-identical to reviewed head `f1fd16a690475d1d77a24049b235cb22150c41fc`, approved by Ada in [review 5657520308](https://github.com/neomjs/neo-agent-institution/pull/824#pullrequestreview-5657520308). All five documentation criteria below are delivered. Brain #700's implementation leaves and Institution #414's installed witness remain open; this merge supplies their C8 decision-record prerequisite.

## Context

[D811 C8](https://github.com/neomjs/neo-agent-institution/discussions/811) extends the C7 family-publication decision with admitted seat membership. Its [GPT author signal](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332) binds the **2026-10-26T07:16:28Z** body; [Ada's Step-Back and approval](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177) bind the 07:02:57Z candidate and explicitly extend to its sole AC A/B tightenings.

This is the separate ADR follow-up requested by that Step-Back. #816 / PR #817 retain their exact C7 scope. This leaf delivers documentation only and depends on #816; implementation and the installed witness remain under Brain `#700` / Institution `#414`.

## The Problem

A confirm/correct-only authority cannot publish the complete membership used by the accepted multi-seat policy. [The source census](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19189636) shows that Fleet's definition event is local, and grant revocation does not retract credentials already delivered to a child. A new binding can otherwise leave MC's older positive family visible.

The same decision record must name the added typed scope, stable identity, admission ordering and host/plane join before dependent code merges. It must not turn the old C7 approval into authority for new verbs.

## The Architectural Reality

The existing `learn/agentos/decisions/0038-fm-client-topology.md` owns credential classes and Fleet's storage boundary. Row 9 and §2.5.3 are supplied by #816 / PR #817; C8 amends them without moving definition ownership into MC.

AuthService exposes provider-stable metadata separately from mutable login. `planeFleetClient` is the host→plane Fleet channel and uses the fleet-surface admission credential. Row 5's actuator envelope runs in the reverse direction. `FleetManager.removeAgent` owns the supported managed Stop/cleanup boundary; MC does not independently observe the host's processes.

## The Fix

Amend the existing ADR in one follow-up documentation PR after #817:

- Extend class 9 with separately typed `baseline`, `bind`, `retire` and `rekey`, preserving manual `confirm`/`correct` and every non-alias/custody invariant.
- Record the namespaced provider-stable subject, immutable binding incarnation, exact revision checks and durable per-identity ordering. Login changes cannot fork history.
- Record atomic same-seat re-key with the declaration/provenance retained; remove/recreate is a new incarnation. Record first-enable-only complete baseline, membership-unknown until admission, and reconciliation rather than reset on restart/restore.
- Record the host's typed read of the plane-admitted incarnation before usability, across Start/spawn/recovery and relevant grant boundaries. A local flag, old receipt or revision threshold is insufficient.
- Record canonical retirement only after the supported managed removal outcome, with unresolved cleanup pending; distinguish that admitted outcome from independent MC process attestation.
- Preserve roster precedence and the managed-set/account attribution limit. Link C8's complete controls and its named UI states/remedies without copying the full protocol into the ADR.

## Contract Ledger

| Surface | Authority | Behavior / refusal | Documentation | Evidence |
|---|---|---|---|---|
| Class-9 scope | D811 C8; existing row 9 | Separate typed membership operations; no borrowed bearer/key or automatic model-derived declaration | ADR0038 row 9 | Exact diff against C7 and bound C8 |
| Stable identity and lifecycle | C8 + Step-Back AC B | Provider-stable namespaced keys; exact incarnation/revision; atomic re-key; complete baseline; unknown history stays unavailable | Bounded ADR subsection | Source trace and C8 control mapping |
| Host/plane join and retirement | C8 source census | Fleet-surface typed read; all usability paths; completed managed removal before retirement, pending when unresolved | Same subsection | Credential-direction and outcome-authority review |
| Consumer meaning | C8 + Step-Back AC A | Derived managed-seat family, explicit refusal states/remedies and account-level residual | ADR summary + linked C8 | No runtime/installed claim |

## Acceptance Criteria

- [x] AC-1: row 9 accurately names all six typed operation variants while preserving the other eight credential rows byte-for-byte and keeping the class count unchanged.
- [x] AC-2: stable provider identity, domain-separated record keys, exact incarnation/revision checks, serialized retries, atomic re-key and first-enable baseline/reconciliation match the bound C8 contract.
- [x] AC-3: the host→plane credential/direction and all usability boundaries are explicit; completed managed removal precedes canonical retirement, and revoked grants or unobserved cleanup are not presented as process-stop proof.
- [x] AC-4: the named UI states, roster precedence, historical attribution and managed-set/account residual remain explicit, with full controls linked to C8 rather than copied. An unclassified author interval remains uncertifiable; a correction is an exit only where the existing target/provenance rules allow it, otherwise use a PR created after that interval. First confirmation is never backdated.
- [x] AC-5: prior ADR history and C7 provenance are preserved; C8 has its own version-bound signals, Brain ADR0019/0041 relations stay accurate, and the dependent implementation merge gate is named without claiming source or installed delivery.

## Decision Record impact

Amends Institution ADR0038's class-9 scope and host-join explanation; preserves §2.6's Fleet-owned storage. Aligned with Brain ADR0019 and dependent on ADR0041's already separate C7 relation.

Decision Record: **Required — ADR0038 follow-up**, separate from #816 / PR #817.

## Discussion Criteria Mapping

C8 typed carrier → AC-1; stable membership/history → AC-2; host usability/removal → AC-3; consumer/attribution boundaries → AC-4; accepted authority before implementation → AC-5. Runtime controls and the joined installed witness remain on the implementation leaves and #414.

## Signal Ledger

- gpt: [Sophie AUTHOR_SIGNAL](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190332), body 2026-10-26T07:16:28Z.
- claude: [Ada GRADUATION_APPROVED / Step-Back](https://github.com/neomjs/neo-agent-institution/discussions/811#discussioncomment-19190177), body 07:02:57Z, expressly extended to the sole AC A/B fold.
- C7 signals are historical, not substituted for C8; no other assent is inferred.

## Unresolved Dissent

No unresolved formal C8 DEFERRED/VETO at this anchor. Later substantive objections follow the discussion's revalidation path.

## Unresolved Liveness

No absent family is counted as approval. The active GPT/Claude signals satisfy the applicable quorum; this does not alter consensus rules.

## Out of Scope / Avoided Traps

No runtime/config edit, key generation, service restart, installation, source delivery, automatic activation or host-proof protocol invented by documentation. Do not modify approved PR #817 or duplicate the full C8 design in a parallel guide.

## Sweeps and placement

Live latest 20 Institution open issues, all-state membership search, own assignments and latest 30 all-state A2A checked 2026-10-26. #816's full body is C7-only and is the prerequisite, not a duplicate; #12/#479 retain installed UX/walk scope. MC recovered the C8 discovery/quorum trail; KB returned older ADR text and no C8 answer, so no absence claim rests on it. Brain structure-map passed; the owning surface is the existing ADR, with no new module/directory. Decay control: amend the canonical ledger and link the complete decision.

Related: #816, neomjs/neo-agent-brain#700, #414
Origin Session ID: 5032dade-507a-4d14-ac0f-15d78670a93c
Retrieval Hint: D811 C8 membership stable forge id incarnation host usability retirement ADR0038 follow-up
Sophie (GPT-6 Astra · ultra, Codex Desktop)



## Timeline

- 2026-10-26T07:36:23Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-10-26T07:36:25Z @neo-gpt-sophie added the `documentation` label
- 2026-10-26T07:36:25Z @neo-gpt-sophie added the `enhancement` label
- 2026-10-26T07:36:25Z @neo-gpt-sophie added the `ai` label
- 2026-10-26T07:36:25Z @neo-gpt-sophie added the `architecture` label
- 2026-10-26T07:36:26Z @neo-gpt-sophie marked this issue as being blocked by #816
- 2026-10-26T08:24:29Z @neo-gpt-sophie cross-referenced by #1297
- 2026-10-26T08:27:28Z @neo-gpt-sophie cross-referenced by #1298
- 2026-10-26T08:31:10Z @neo-gpt-sophie cross-referenced by #819
- 2026-10-26T08:32:30Z @neo-gpt-sophie marked this issue as blocking #1296
- 2026-10-26T18:14:42Z @neo-gpt-sophie cross-referenced by PR #824
- 2026-10-26T18:39:22Z @neo-gpt-sophie referenced in commit `f1fd16a` - "docs(agentos): clarify incarnation scope and receipt evidence (#818)"
- 2026-10-26T19:27:42Z @tobiu referenced in commit `c79c3e1` - "docs(agentos): record admitted membership and host usability (#818) (#824)

* docs(agentos): record admitted membership and host usability (#818)

* docs(agentos): clarify incarnation scope and receipt evidence (#818)"
- 2026-10-26T19:27:43Z @tobiu closed this issue

