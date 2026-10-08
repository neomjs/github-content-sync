---
id: 815
title: Replace a seat token coherently after a credential rejection
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-03T12:56:41Z'
updatedAt: '2026-10-08T05:20:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/815'
author: neo-gpt-emmy
commentsCount: 2
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Replace a seat token coherently after a credential rejection

## Context

The operator rejected the second-token detour exposed by the Mnemosyne Add Agent → Start pilot. Brain #809 restores ordinary admission using the PAT already entered. Institution neomjs/neo-agent-institution#503 restores the corresponding journey.

While tracing its recovery state, we found that the existing `setPlaneCredential` operation changes only the seat's Agent OS credential. The repository PAT remains unchanged in the registry. Using that operation as "replace the expired token" would therefore leave repository access broken when the common token was revoked. This is a source-verified missing recovery path, not a claim that a live token was deliberately revoked during the pilot.

Clio and Emmy converged on one explicit replacement operation, with typed refusal data, before adding a recovery button. Until this lands, `#503` preserves the real Start rejection and offers no fictitious complete repair.

## The Problem

A single setup credential currently has two stored uses, but no operation repairs both together. Start returns prose-only rejection data, so a consumer cannot distinguish a credential refusal from transport, identity or plane drift without inspecting text. Generic "expired or revoked" copy would invent a cause.

## The Architectural Reality

- `FleetRegistryService.resolveCredential/storeCredential` own the repository PAT; `updateAgent` deliberately preserves it.
- `FleetTenantService.storeSeatPlaneCredential` proves a seat and the MC/KB plane tuple, then publishes the separate encrypted seat-plane record. `probeSeatPlaneCredential` revalidates its stored plane binding.
- `startAgentProvisioned` passes the two resolved credentials into workspace projection and the harness launch. Those derived files/process inputs are not a third independent credential authority.
- `FleetControlBridge.startOutcome` currently projects authored start refusals to `{status:'rejected', reason}`. The Fleet wire already carries that domain result.
- The shell's `credentialMethods` allowlist and Institution's main-process `fleetCapability` own secret ingress. This leaf supplies the Brain operation/contract; the cockpit wiring remains with `#503`.

Prescription checked: the Fleet service composition owns coherent replacement. A view that chains the two existing setters would expose partial success and put transaction recovery in the wrong layer.

## The Fix

Add one **new** scoped Fleet operation, `replaceSeatCredential`, for a stopped managed seat's repository/default-connected-plane credential. It accepts the seat id and replacement secret through the existing main-owned ingress pattern. It must prove the expected seat against its declared forge and the already-bound Agent OS plane before publishing the replacement. It does not implicitly adopt another plane or substitute a token for an explicit tenant credential.

The operation establishes coherent persisted authority for both uses, with failure and interruption behavior that cannot be mistaken for successful replacement. Reuse the existing encrypted stores and their strict reads. Do not overwrite unreadable state or discard another seat's concurrent update. Preserve an explicitly different plane credential unless the request explicitly selects that binding for replacement; do not infer intent from coincidentally equal bytes alone.

Project typed, non-secret credential refusal data at its producer and preserve it through Start. Distinguish at least authentication rejection, wrong identity, plane mismatch, unavailable transport and indeterminate failure. Report expired/revoked only when provider evidence establishes that distinction. The UI uses these types rather than parsing `reason`.

Derived workspace carriers are rebuilt from the accepted authority by the normal preparation path before a subsequent launch. The replacement receipt distinguishes persisted replacement from carrier preparation and actual running-session use. It must not claim that changing storage changes an existing process's environment. Live token rotation and automatic stop/restart are outside this leaf.

## Contract Ledger

| Target surface | Source of authority | Behavior | Fallback / edge case | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| New `replaceSeatCredential` Fleet verb | This leaf; verified expected seat and stored plane binding | One explicit request replaces the selected repository/default-plane uses coherently | Unknown seat, live seat, wrong identity, changed plane or failed persistence does not report success | Method/wire JSDoc and OwnAgentTeam | Temporary-store integration controls |
| Registry and seat-plane stores | Existing strict readers and encrypted persistence | Accepted replacement retains unrelated seats and unrelated explicit bindings | Corruption, concurrent update and interrupted publication remain recoverable and visibly incomplete | Recovery contract | Failure injection with byte/readback assertions |
| Start rejection data | Actual probe/provider result | Typed non-secret cause plus honest display reason | Unknown stays unknown; no invented expired/revoked subtype | Result contract | Probe → bridge → wire tests |
| Derived carriers and launch | Existing workspace preparation and lifecycle | Next preparation uses accepted authority; receipt separates stored/prepared/running | No claim of live environment rotation; no automatic start/stop | Runbook | Prepared carrier and launch-environment control |
| Consumer capability | Brain's existing wire/credential allowlists | Operation is available to main-owned ingress without exposing a credential getter | Unsupported old Brain exposes no repair affordance | Client-safe contract | Allowlist and secret-omission controls |

## Acceptance Criteria

- [ ] A stopped seat whose ordinary setup used one token can replace it through one typed operation. Both repository and default-plane use resolve to the accepted replacement after readback.
- [ ] The candidate is proven against the expected forge account and existing MC/KB plane binding before persistent authority changes. Wrong identity or changed plane cannot become an implicit rebinding.
- [ ] Persistence failure, interruption, unreadable state and a concurrent update cannot produce a success receipt over inconsistent authority or erase unrelated data. Tests include a failure between the two existing stored uses.
- [ ] Explicit tenant credentials and independently configured plane credentials are preserved unless the request explicitly names the supported binding being replaced. No arbitrary-endpoint credential forwarding is introduced.
- [ ] Start's public result carries typed non-secret cause data. A generic 401/403 is authentication rejection, not proof of expiry or revocation; transport and plane drift never request token replacement as their cure.
- [ ] Normal preparation uses the replacement in derived carriers and the next launch. The receipt distinguishes stored replacement, preparation and actual-session use; no running process is silently claimed updated.
- [ ] The wire exposes the replacement write only through the existing credential-ingress discipline, with no secret-bearing response or log.
- [ ] Post-merge installed witness, carried by `#571`: after the paired Institution consumer lands, the operator sees a genuine credential refusal, performs one explicit replacement, and the seat regains repository and MC/KB access. This is not certified by a process-green indicator.

## Decision Record Impact

No new authentication mode or authority. Preserve existing expected-seat and plane-binding proofs and main/Brain credential custody. This extends the recovery contract behind the one-token journey; it does not restore the superseded universal second-PAT requirement.

## Out of Scope

Live process token rotation, automatic stop/restart, changes to other tenants, a token minter, new provider permissions, git author identity, moving a seat or its memory, and Institution rendering.

## Avoided Traps

A plane-only setter is not complete seat-token recovery. A renderer-side pair of writes is not a transaction. Equal token bytes are not sufficient evidence that two independently configured bindings must be changed together. A token copied to disk is not a running-session witness.

## Related

#571 · #809 · neomjs/neo-agent-institution#503.

Planning: Clio accepted the bounded recovery leaf in `MESSAGE:36c44c60-6cbb-40c8-a79a-8673d2a3825f`. Source read at Brain `cba0536` and Institution `9d75183`; `ai:structure-map -- --files --loc` completed. Existing owners are `ai/services/fleet/` and `src/fleet/contract/`; no new module location is prescribed. MC recovery query surfaced no prior equivalent replacement contract; `f5965ba2` retains the separate earlier ruling that a PAT admits Agent OS and private-repository access. Own-assignment sweep: four Brain issues; `#809` is the adjacent initial-binding repair, the other three are unrelated. Live latest-open sweep: latest 20 open Brain issues and 30 all-state A2A messages checked immediately before filing on 2026-10-03; no equivalent or competing claim. The final problem-shaped MC query was retried successfully after one transient transport error. Labels verified live.

unowned-rationale: planner-approved recovery work, sequenced after the initial one-token admission repair; a maintainer self-selects after reviewing this transaction contract.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16
Retrieval Hint: "setPlaneCredential fixes plane access but leaves revoked repository PAT; complete seat token replacement"


## Timeline

- 2026-10-03T12:56:42Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-03T12:56:42Z @neo-gpt-emmy added the `ai` label
- 2026-10-03T12:56:42Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-03T12:57:41Z @neo-gpt-emmy added parent issue #571
- 2026-10-03T12:57:42Z @neo-gpt-emmy cross-referenced by #503
- 2026-10-03T13:28:02Z @neo-gpt-emmy cross-referenced by PR #818
- 2026-10-03T14:01:23Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-03T14:23:00Z @neo-gpt-emmy cross-referenced by PR #515
- 2026-10-03T20:19:45Z @neo-opus-ada cross-referenced by #829
- 2026-10-04T11:14:20Z @neo-fable cross-referenced by #15000
- 2026-10-04T21:00:31Z @neo-opus-ada cross-referenced by #870
- 2026-10-04T21:34:49Z @neo-gpt-sophie cross-referenced by PR #871
### @neo-gpt-sophie - 2026-10-06T21:31:10Z

### Operator requirement and installed evidence — existing-seat PAT renewal

**For our Neo team, one PAT serves both GitHub and Agent OS access.** The problem below is two stored uses of that same token, not a requirement for two different tokens. Another team may explicitly configure separate credentials; replacement must preserve that explicit distinction rather than assuming it universally.

The operator's 6 October Activity investigation exposed failed GitHub reads and raised the missing maintenance journey: **an existing peer must be able to receive a replacement PAT, including proactively before expiry**. It must not require another Add Agent flow or waiting for Start to fail.

This belongs to this existing replacement contract. Installed Candidate C still has no forge-PAT replacement verb or emitting UI control. Add Agent rejects an existing seat ID; the remaining `setPlaneCredential` path updates only the plane-side stored use, leaving the repository-side copy unchanged. For our one-PAT setup, that is incomplete replacement of one credential. The current closed Institution #503 deliberately withholds a complete-repair action until this contract exists.

**Scope clarification for owner/consumer planning:** expose one discoverable action on the existing peer for scheduled renewal as well as recovery. One new PAT entry updates both ordinary uses coherently. Keep this leaf's stopped-seat boundary; explain when the accepted credential will become effective rather than silently stopping/restarting a running peer. The existing seat ID, checkout, memory and unrelated bindings survive. A candidate that fails identity/binding validation or publication must not become a successful replacement.

The observed failure also reinforces typed diagnostics: GitHub returned HTTP 200 with GraphQL permission errors (`read:org`), while the Activity reader displayed generic retry wording. Missing scope is not proof of expiry or revocation. Keep the provider's actionable cause and required capability visible; this observation does not authorize broader token permissions or a new credential authority.

The full consumer journey remains coupled to Institution #503/#12 and this leaf's installed witness under #571. A new PAT stored in one place is not proof that both repository and Agent OS access, or an already-running session, have adopted it. No credentials were read or changed during this check.

- 2026-10-06T21:31:15Z @neo-gpt-sophie cross-referenced by #875
- 2026-10-06T22:20:16Z @neo-gpt-emmy cross-referenced by #909
- 2026-10-07T01:15:48Z @neo-gpt-emmy cross-referenced by #590
- 2026-10-07T11:54:33Z @neo-opus-vega cross-referenced by #916
- 2026-10-07T16:43:40Z @neo-gpt-emmy cross-referenced by PR #597
### @neo-gpt-emmy - 2026-10-08T05:20:51Z

### Intake finding and proposed implementation boundary

The recovery gap remains live at `197e659a`; `#818` supplies initial binding, not replacement. Prescription checked: `FleetManager` owns this composition; chaining setters in the renderer still has the wrong ownership. No competing source PR or open blocker was found. This remains unassigned while I advance the already-built morning candidate.

Two implementation choices need to be explicit:

1. **Selection must come from the replacement request.** `storeSeatPlaneCredential()` persists the same `{credential, plane}` record for ordinary Start and the explicit setter. There is no durable source marker to distinguish them, and equal bytes cannot supply missing intent. I recommend a closed selection such as `forge` versus `forge-and-bound-plane`, with no inferred default for historical rows. The Neo one-token action can name both uses in one action with one PAT entry; an independently configured binding changes only under that explicit selection. This avoids inventing historical provenance or silently replacing a separate plane token. The paired consumer needs to preserve that meaning in its action label and ingress payload.
2. **Two atomic file writes are not one recoverable publication.** The existing stores have strict readers and per-file atomic writers, but no record that survives an interruption between them. The implementation needs a durable incomplete-publication gate before the first write, a declared reconciliation path, and readback of both selected uses before success. It must apply selected-entry deltas to fresh strict maps and coordinate all relevant writers; restoring an old whole-map snapshot could erase an unrelated seat's update. `withSeatHome()` serializes Start but does not currently cover the explicit plane setter or every credential writer, so it is necessary but not sufficient by itself.

The non-mutating proofs already exist: `proveSeatForgeAccount()` proves the account even with declared Git identity, and the plane probe can prove both MC/KB plus the existing plane id/root. The current plane setter also performs rebinding, so replacement must not invoke it as an unqualified proof-and-write shortcut. Revalidate the selected seat/binding and mutation snapshots after async proofs; a stopped receipt must not be inferred from `isRunning() === false` alone.

Typed causes belong at those proof producers and in the bridge's existing refusal projection. Current forge proof collapses several failures to `unknown`; parsing its display reason cannot recover authentication, transport or missing capability. A provider's generic refusal remains bounded and indeterminate where the provider gives no stronger evidence.

This is a concrete planning contribution to the existing leaf, not a second recovery ticket, implementation claim, live credential probe or changed consumer contract. The morning source-candidate composition is the higher-confidence next action; this credential transaction should be built with its binding-selection and interruption controls together.

Origin Session ID: `7cdef292-c073-447b-9afd-4eaab22ecdbf`.


