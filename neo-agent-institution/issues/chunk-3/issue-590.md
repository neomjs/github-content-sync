---
id: 590
title: Explain native tool launch admission on the seat card
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-07T01:15:47Z'
updatedAt: '2026-10-07T17:33:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/590'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 909 Launch Desktop MCPs through scoped Fleet admission'
blocking: []
closedAt: '2026-10-07T17:33:46Z'
---
# Explain native tool launch admission on the seat card

## Context
The accepted Desktop-profile launcher design requires a visible recovery surface. Brain PR neomjs/neo-agent-brain#910 publishes `launchAdmission`, but Institution's current roster model and seat card do not consume it. [The row-2 handoff](https://github.com/neomjs/neo-agent-institution/issues/477#issuecomment-6028232910) already retains this obligation.

Design authority: [D19437](https://github.com/neomjs/neo/discussions/19437), merged [ADR 0038 §2.5.2](https://github.com/neomjs/neo/blob/b2db92c5b74848f520d8cf23095ace3bf1b6761a/learn/agentos/decisions/0038-fm-client-topology.md), and `#477`'s reason/next-step outcome. The ADR says: “The seat card renders issuer-owned typed outcomes and a managed-restart action for stale new-child admission”. This is the planned source consumer, not a new outcome epic or an installed row-2 pass.

## The Problem
After an issuer restart or grant revocation, a running Desktop can retain working tools while new children cannot start. The card must explain that distinction and expose the appropriate existing action. A credential that cannot be proved is a different problem: restarting does not repair its owner/value.

## The Architectural Reality
At Institution `3b68995f` (same tree as the mapped `85d5282` candidate), `apps/agentos/model/FleetAgent.mjs` has no `launchAdmission` field. `apps/agentos/view/fleet/roster/card/Container.mjs` renders lifecycle/session-folder feedback and emits existing lifecycle intents; `apps/agentos/util/FleetLifecycleIntentAdapter.mjs` maps those intents to the bridge.

The new producer contract is [src/fleet/contract/launchAdmission.mjs](https://github.com/neomjs/neo-agent-brain/blob/edc0c7eafe7c9a770418d01281bed5d4185661b1/src/fleet/contract/launchAdmission.mjs), exported through `neo-agent-brain/fleet-contract`. States are `none/reserved/active/revoked/stale`; revocation reasons, refusal codes and credential-owner labels are distinct fields. The relevant credential refusal codes are `credential-missing` and `credential-unproven`, with owner `seat-pat` or `plane-bearer`.

## The Fix
Pin reviewed Brain #910 merge `2d839fc1b0a191d4dcfde35f3bd95ea3728d39d3`, admit its public module through the packaged renderer's exact asset list, carry its typed field through the existing roster projection/model and render concise seat-card guidance using the current lifecycle-intent path. Import the public contract; do not import `ai/` or derive authority in the client.

- For a running seat with `stale` or `revoked` admission, describe admission for **new tool connections** and offer the existing managed restart when its lifecycle controls admit that action. Preserve stopped/unauthorized/pending-action guards. Do not claim already-running tools disconnected.
- For a recent credential refusal, name the affected credential owner in user language and provide the truthful supported next step. For per-server revocation inside an active generation, use the same roster row's public `agent` definition (`forge` and sparse `mcpServers`) through the canonical `mcpCatalogFor` / `resolveMcpMatrix`: intentionally disabled servers stay quiet; currently enabled, revoked grants for `server-disabled` or `plan-changed` may recommend the admitted Restart. A startup `credential-missing` grant is not a restart cure. Missing intent does not imply enablement. Keep the generic Restart control under its normal lifecycle rules, but do not present restart as a credential repair, expose values/env-slot jargon, or send the operator to an unrelated credential store. Reuse an owner-correct existing settings/recovery surface; where none exists, state the unsupported boundary and retain the already-open recovery obligation rather than invent a writer.
- `reserved` is preparation; `active` is admission, not proof of tool connectivity. `none` is the producer's explicit no-generation state. Missing, unknown or failed source data must not become active/healthy by a default.
- Retain current generation/source semantics when choosing recent refusal feedback. An old refusal must not survive a successful newer observation or a seat/source switch as a current diagnosis.

A current `active` generation can also contain a recent credential refusal for one server. Render that failed new-child attempt without changing the seat's running fact or claiming other tools failed. Preserve absent/old-Brain null and the runtime-source freshness gate; fold guidance into the existing `pendingAction` / `controlReason` / `sessionFolder` priority instead of adding a second status writer.

The card binds the existing provider's `gridAdapterState` and requires observed runtime provenance, so retained rows do not assert current admission. The bounded audit is generation-scoped; each server's newest outcome supersedes its earlier attempts. Keep presentation wording in a `SeatLaunchAdmission` sibling of `SeatModel` / `SeatSessionFolder`, within the existing model, card and intent/controller seams. No new panel, provider island or parallel restart mechanism.

## Contract Ledger
| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Roster projection and `FleetAgent` model | Public Brain `launchAdmission` contract | Preserve typed admission data and the same-seat current public MCP intent needed by the card | Missing/invalid stays unknown; clear incompatible retained state | Field/source docs | Production projection/model controls |
| Seat-card admission guidance | Same contract; `#477` | Running-seat `stale/revoked` and currently enabled revoked server grants offer an admitted managed restart; stopped seats and intentionally disabled servers stay quiet | Preserve existing lifecycle action guards and already-running-tool distinction | State/next-step census | Fixture render and click/dispatch controls |
| Credential refusal guidance | Closed refusal/credential vocabularies | Name seat or plane credential without values; use the supported owner-correct next step | No restart cure or fabricated credential editor | Same census; retain neomjs/neo-agent-brain#815's separate recovery scope | Both credential-owner cases and no-restart negative control |
| Candidate dependency | Reviewed Brain producer revision | Manifest/lock pin and package consume the same contract | Do not claim installed delivery from the pin | Candidate receipt on `#12` | Pin/lock validation and source fixtures |

Decision Record: Required — ADR 0038 amendment merged in neomjs/neo#19439.

Decision Record impact: aligned with ADR 0038 class 8; consumes its existing projection. No new authority or credential-use contract.

## Acceptance Criteria
- [ ] A merged/reviewed Brain revision containing `#910` is pinned coherently and the public contract is imported through the supported export.
- [ ] Real roster projection/model data reaches the card, including update and source/seat-change paths; no fixture-only parallel field path.
- [ ] Stale/revoked guidance distinguishes new-child admission from already-running tools and invokes the existing managed restart exactly once when allowed.
- [ ] Both credential refusal codes identify the correct owner, expose no secret/argv/raw error, and offer no restart as the credential remedy.
- [ ] None/reserved/active and missing/unknown/failed data preserve their distinct meanings; no connectivity claim follows solely from active admission.
- [ ] A newer valid observation or a seat/source change removes obsolete diagnostic/action state; stale retained data is not shown as a current proof.
- [ ] Fixture render/action controls exercise the actual mapper → FleetAgent → card → intent chain: denied/pending actions, stale/revoked, absent snapshots, and runtime wired/observed + running seat + active admission + recent credential-unproven for a named server. The last case shows the refusal without promising restart will repair it. A same-card transition to a replacement or absent snapshot clears earlier warning/restart guidance.
- [ ] Update the existing state/next-step census or source guidance with the measured behavior and remaining installed witness.

## Post-Merge Validation
`#477` and [`#12`'s candidate record](https://github.com/neomjs/neo-agent-institution/issues/12#issuecomment-5991701878) retain the installed card/recovery witness on the exact package. neomjs/neo-agent-brain#571 retains Desktop-profile availability, native Code connectivity, and correct identity/plane read-write. This leaf does not close those outcomes.

## Out of Scope
Brain issuer/lifecycle repair, a new credential-replacement writer (neomjs/neo-agent-brain#815), generic Stop cancellation, Activity visibility policy (D19440), and live installation/seat restart during source work.

## Related and sequencing
Parent: #477. Candidate: #12. Producer: neomjs/neo-agent-brain#909 / PR neomjs/neo-agent-brain#910.
BLOCKED_BY neomjs/neo-agent-brain#909

## Filing evidence
Live source and the row-2 owner handoff confirm the missing consumer; no equivalent issue/PR was found. Own-assignment sweep: only `#42`, an investigation contract, not this source leaf. MC recall recovered the current candidate/acceptance boundaries; source and peer receipts determine current behavior. Placement is existing `apps/agentos` model/card/controller code, so no new Agent OS source directory/file is prescribed. Live latest-open sweep: latest 20 open issues checked at 2026-10-07 01:15:19 UTC; no equivalent found. Recent all-status A2A claim sweep found no competing author. MC problem queries support the source/consumer separation; no superseding prior decision surfaced.

Origin Session ID: d0d0bed3-7ce4-4bce-a16d-59589484aec0
Retrieval Hint: launchAdmission seat-card consumer; credential refusal without restart; stale/revoked new-child admission.




## Timeline

- 2026-10-07T01:15:47Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-07T01:15:48Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-07T01:15:48Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-07T01:15:48Z @neo-gpt-emmy added the `ai` label
- 2026-10-07T01:15:48Z @neo-gpt-emmy added the `design` label
- 2026-10-07T01:16:04Z @neo-gpt-emmy added parent issue #477
- 2026-10-07T01:16:29Z @neo-gpt-emmy marked this issue as being blocked by #909
- 2026-10-07T01:17:07Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-07T01:17:08Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-07T01:21:34Z @neo-gpt cross-referenced by #477
- 2026-10-07T12:50:28Z @neo-gpt-emmy cross-referenced by #591
### @neo-gpt-emmy - 2026-10-07T16:23:05Z

Intake: **valid-as-written**, with source freshness made explicit in the card binding. The live blocker Brain #909 is closed; Institution `dev@46929bee` still has no `launchAdmission` mapper/model/card path. The drift probe intersected package files (cssnano update), so I ran the full intake rather than using the earlier-session exemption. Created/updated 2026-10-07T01:15:47Z; no stale/exemption label. The canonical inactive workflow uses 90+14 days, but Institution does not host that workflow; no bot band is treated as currency evidence.

Prescription checked independently: `RosterRow.mapRosterRow` / `FleetAgent` own the nullable DTO fact, and `roster/card/Container.applyRecord` owns its guidance. `FleetLifecycleIntentAdapter` remains the existing intent transport. A same-profile read failure retains records, so admission guidance must consume the existing provider `gridAdapterState` as well as runtime provenance; it must not treat a retained refusal as current. Wrong-folder, pending action and existing refusal feedback keep priority. Credential refusal may outrank only the ordinary pending/unknown folder fallback.

Public Brain contract verified at merged #910 revision `2d839fc1b0a191d4dcfde35f3bd95ea3728d39d3`; current `a8dd1ae4` pin lacks it. Minimal producer pin chosen. Credential recovery remains Brain #815: no seat-PAT editor currently exists, and plane-credential ownership is distinct. No new operator obligation, credential writer or live restart is introduced.

ADR successor-risk: **adr-aligned** — current ADR 0038 §2.5.2 requires this exact card/recovery consumer; #19453 changes content observation, not native launch admission. Independent parent epic review: https://github.com/neomjs/neo-agent-institution/issues/477#issuecomment-5967397891. MC prior-art `bb50e1ca-cbe4-4c20-8dd6-ffc8efeadca6` preserves the active-generation + failed-new-child counterexample; KB returned adjacent, not equivalent, work. Missing graph pre-brief was `NODE_NOT_FOUND`, so live ticket/source plus this MC record supply the brief. No open Institution PR or same-scope claim found.

Core idioms read: Engine `Neo`, `Base`, `state.Provider`, `data.Model`, `data.Store`; updates remain through the existing shared record/provider path. ROI: closes an already-planned FM state/next-step gap without adding authority or another status writer.

- 2026-10-07T16:43:40Z @neo-gpt-emmy cross-referenced by PR #597
- 2026-10-07T17:23:13Z @neo-gpt-emmy referenced in commit `9ecb8e4` - "fix(fleet): qualify launch admission guidance by current seat intent (#590)"
- 2026-10-07T17:33:46Z @tobiu referenced in commit `fd958fb` - "feat(fleet): explain native tool launch admission (#590) (#597)

* feat(fleet): explain native tool launch admission (#590)

* fix(fleet): qualify launch admission guidance by current seat intent (#590)"
- 2026-10-07T17:33:46Z @tobiu closed this issue

