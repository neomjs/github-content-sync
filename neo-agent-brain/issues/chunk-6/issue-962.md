---
id: 962
title: Preserve observer-scoped mailbox reads in Fleet Activity
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-10-09T23:24:38Z'
updatedAt: '2026-10-10T12:26:02Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/962'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 414
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 596 Show All / involves-me A2A activity in Fleet'
closedAt: '2026-10-10T12:26:02Z'
---
# Preserve observer-scoped mailbox reads in Fleet Activity

## Context
Institution's All / involves-me Activity journey (neomjs/neo-agent-institution#596, owned by Grace) needs the canonical observer read delivered by #921. [The consumer intake](https://github.com/neomjs/neo-agent-institution/issues/596#issuecomment-6090778838) found that Activity drops its observer request. Independent source inspection at `16b027880dab3f15bb60c1349a77e2c8e31a8891` confirms the gap.

## The Problem
`wireFleetActivityReadSource.mjs#makeReadA2ASnapshot` forwards only limit/offset to ordinary `listMessages`. Explicit observer scope never reaches the admitted mailbox reader. The A2A adapter can interpret a structured refusal without messages as an empty array; the composer also discards observer admission and page metadata. Merely forwarding a parameter would therefore leave false-empty and retention hazards.

## The Architectural Reality
Design authority: [ADR 0038 §2.3.1](https://github.com/neomjs/neo/blob/87b051d0235de76e8685bd0843e5516a13754006/learn/agentos/decisions/0038-fm-client-topology.md#231-the-explicit-team-a2a-observation-d19440): “One eligibility set governs admission, count, page and detail.” It also requires non-stamping observation, unchanged ordinary inbox defaults and truthful failures.

The existing owners are `ai/services/fleet/{FleetControlBridge,wireFleetActivityReadSource,fleetA2AActivityAdapter,fleetActivityComposer}.mjs`. Reuse `normalizeMailboxObserver`, server-bound `observeMessages`, and the plane client's connected-schema gate. Memory Core retains policy and population authority.

The composer currently retains the first ordinary mailbox page in viewer-bound `heldA2A` and folds it into persisted lane claims. An explicit expanded observer read must not change that background source. Activity offsets already belong to A2A only; history requests select `slots: ['a2a']`.

## The Fix
Wire the existing observer reader through local and plane Activity composition. Preserve canonical, A2A-qualified admission/page metadata through the adapter and composer, with explicit refusal handling. Keep ordinary reads and their held snapshot/lane fold unchanged; exclude explicit observer pages from that cache. Update the existing owners and their focused specs, without a new service or policy engine.

## Contract Ledger
| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `fleetActivity` explicit observer | ADR0038 / #921 / existing normalizer | Server-bound observer read in both modes; closed scope and supported bounds | Invalid selectors or unsupported plane fail closed; no ordinary-read fallback | Bridge/wiring JSDoc | Local and plane composition controls |
| A2A contribution/page | Canonical mailbox envelope | Preserve `observation` and filtered `totalCount,truncated,nextOffset,limit,offset`, qualified to A2A | Malformed/refused answer is degraded, never healthy empty | Adapter/composer envelope contract | Later-page, clamp, empty and refusal controls |
| Ordinary held snapshot/lane claims | Existing composer contract | Explicit observer reads neither replace nor fold into ordinary cache | Existing ordinary behavior remains | Composer retention JSDoc | Interleaved and delayed observer/ordinary reads |

Decision Record impact: aligned-with ADR0038 §2.3.1; no policy amendment.

## Acceptance Criteria
- [ ] **AC-1:** Explicit observer requests reach the canonical non-stamping reader in local and plane modes. Reuse closed observer validation and rejection of caller-supplied viewer/recipient selectors. Omitted observer retains existing behavior.
- [ ] **AC-2:** Preserve the canonical A2A observation context and paging fields through Activity. Continuation reads use the existing A2A-only slot contract; a globally truncated mixed feed must not advertise a continuation that skips omitted mailbox rows. Demonstrate an authorized match beyond page one with interleaved PR events.
- [ ] **AC-3:** Missing observer wiring, unsupported plane schema, structured refusal and malformed envelopes produce an honestly degraded A2A source. Other valid Activity sources remain available. Successful empty, clamped and unavailable answers remain distinct.
- [ ] **AC-4:** Observer Activity reads stamp no seen/read receipt and move no Task in either mode. They do not replace `heldA2A`, advance its ordinary-read generation, or fold into persisted lane claims, including delayed/interleaved completions.
- [ ] **AC-5:** Focused production-composition regression controls exercise AC1–4 and preserve ordinary Activity paging, PR composition and roster lane behavior. Reuse Memory Core policy tests; do not duplicate its eligibility query in Fleet.

## Out of Scope
Institution UI/detail implementation, new sharing policy, client-side mailbox/PR pagination merge, background team activity caches, credentials, deployment and installed acceptance. The named-candidate busy-population journey remains neomjs/neo-agent-institution#490 under #414.

## Avoided Traps
Do not filter one fetched global page for involvement, confuse source count with displayed-event count, turn a refusal into zero traffic, widen an ordinary read implicitly, or persist observer pages as roster lane truth.

## Related and ownership
Producer prerequisite for neomjs/neo-agent-institution#596, within neomjs/neo-agent-institution#414. Reuses closed #921 / PR #952; this remaining Activity integration does not reopen the canonical policy implementation.
unowned-rationale: Filing the agreed prerequisite while the filer handles the release-blocking Dock investigation and owed review; no producer implementation is claimed. Grace retains #596 and can self-select this prerequisite or coordinate its pickup.

## Creation checks
Latest20 open Brain issues (created-descending), recent30 all-state A2As, exact `fleetActivity` history and six own assignments checked 2026-10-09 23:25Z; no equivalent producer issue/claim found. Closed #634 concerns offset forwarding, not observation. KB returned older Activity architecture; the three-result MC symptom sweep was noisy, so neither is treated as absence proof. Current source, ADR and Grace's independently confirmed intake determine the shape. Scoped Brain structure map confirms the existing Fleet owners; no new module prescribed.

Origin Session ID: b56dbc41-6e95-4210-a2ea-8d1f5f3ffcd0
Retrieval Hint: "fleetActivity All involves-me lost observer structured refusal heldA2A laneClaims"


## Timeline

- 2026-10-09T23:24:39Z @neo-gpt-emmy added the `bug` label
- 2026-10-09T23:24:40Z @neo-gpt-emmy added the `ai` label
- 2026-10-09T23:24:40Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-09T23:25:29Z @neo-gpt-emmy marked this issue as blocking #596
- 2026-10-09T23:25:29Z @neo-gpt-emmy added parent issue #414
- 2026-10-09T23:25:30Z @neo-gpt-emmy cross-referenced by #596
- 2026-10-09T23:26:31Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T23:33:10Z @neo-opus-grace unassigned from @neo-opus-grace
- 2026-10-09T23:40:20Z @neo-gpt assigned to @neo-gpt
- 2026-10-10T00:07:57Z @neo-gpt referenced in commit `2204797` - "feat(fleet): preserve observer-scoped Activity reads (#962)"
- 2026-10-10T00:08:20Z @neo-gpt cross-referenced by PR #963
- 2026-10-10T01:27:54Z @neo-gpt referenced in commit `c54d5c4` - "ci(fleet): trigger required CodeQL analysis (#962)"
- 2026-10-10T12:26:03Z @tobiu referenced in commit `be7181b` - "feat(fleet): preserve observer-scoped Activity reads (#962) (#963)

* feat(fleet): preserve observer-scoped Activity reads (#962)

* ci(fleet): trigger required CodeQL analysis (#962)"
- 2026-10-10T12:26:03Z @tobiu closed this issue

