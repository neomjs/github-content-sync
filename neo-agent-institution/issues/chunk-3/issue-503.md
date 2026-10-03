---
id: 503
title: Add Agent supplies everything needed for Start
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-opus-ada
  - neo-gpt-emmy
createdAt: '2026-10-03T11:48:34Z'
updatedAt: '2026-10-03T18:44:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/503'
author: neo-gpt-emmy
commentsCount: 3
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
milestone: FM v1
---
# Add Agent supplies everything needed for Start

## Context

The operator's 2026-10-03 Mnemosyne onboarding attempt accepted a PAT in Add Agent, then demanded a second identity-only PAT through a button in Agent Details. The operator explicitly rejected that contract. The adjacent URL choice also reverted immediately: live readback showed that its saved credential was already assigned to Sophie.

The [authority correction](https://github.com/neomjs/neo-agent-brain/issues/699#issuecomment-5968691394) records the unapproved credential burden. neomjs/neo-agent-brain#809 owns admission to the connected plane using the PAT already supplied. This leaf owns a coherent Add Agent → Start experience, not just replacement wording on the credential button.

**Design status:** Clio accepted the journey in `MESSAGE:ae0dc9aa-c22d-486e-ba5c-79bc75c9d88d`; her wording, focus and installed-witness requirements are folded below. Implementation can proceed. neomjs/neo-agent-brain#809 must merge before the Institution change's required Brain pin; its PR names that exact pin. No installed-seat change is authorized by this planning update.

## The Problem

Add Agent does not communicate or collect everything that ordinary Start needs. The operator must discover implementation-specific target and credential controls in Agent Details, then encounters a second token request and a URL choice that the registry rejects. Registration therefore looks like completed setup while required input is still hidden elsewhere.

## The Architectural Reality

- The connected Fleet plane is already known. `This fleet` and a saved URL can describe the same endpoint while selecting different credential authorities; these are not interchangeable identities.
- neomjs/neo-agent-brain#809 supplies the existing default-plane proof-and-store path from the seat's already stored PAT. Explicit tenant credentials remain a separate authority.
- `AgentConfigComponent.createTargetChoices/onCardClick` currently offer connected tenants without checking whether another seat owns their saved credential. Brain `FleetRegistryService.configureAgent` deliberately rejects that reuse.
- The root Provider owns the public AgentDefinitions and FleetTenants Stores. `ConfigIntentRoundTrip` writes the selected definition only after an accepted response and retains its race guards.
- `harness/credentialPrompt.mjs` owns secret input in the main process. Credentials do not belong in renderer records or messages.

## The Intended Journey

1. **Add Agent collects the required input in one flow.** Retain the existing identity, harness and repository inputs. For the ordinary connected-plane case, enter the seat's PAT once. Show the connected Agent OS as a clear destination, rather than two competing badges for the same URL.
2. **The normal path has no plane selector or credential-set button.** A newly added seat does not require a visit to Agent Details to supply additional setup before Start. Connection editing is an explicit secondary operation; it must not hide a prerequisite for the normal path.
3. **Start uses what Add Agent collected.** With neomjs/neo-agent-brain#809 present, the ordinary connected-plane flow does not ask for another PAT. If a supported connection genuinely requires different credentials, the same Add flow must explain and collect that requirement before claiming setup is complete. Do not add a new connection mode in this leaf.
4. **A credential request at Start is recovery.** The complete replacement operation is owned by neomjs/neo-agent-brain#815: the existing plane-only setter cannot repair repository access. Until that typed operation exists, this leaf preserves the observed rejection and offers no misleading complete-repair action. Existing prompt wording must not prescribe a second identity-only PAT or categorically forbid the existing seat PAT.
5. **Retained connection choices are usable and truthful.** A saved tenant credential assigned to another seat is not actionable. Explain why if it is shown; preserve the current assignee's choice. Refresh availability from the shared Stores. A concurrent reassignment can still be refused by Brain with a visible reason.

The failed pilot's session-folder, memory and hook findings remain under neomjs/neo-agent-brain#571. One-PAT admission does not certify a usable Claude session. This leaf must not imply that process readiness proves those effects.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / recovery | Evidence |
| --- | --- | --- | --- | --- |
| Add Agent → Start | Operator correction; connected plane; neomjs/neo-agent-brain#809 | One PAT entry in the ordinary flow; no required detour through Agent Details | Necessary exceptional input is explained in the same flow | Complete journey on the installed candidate |
| Primary configuration | Existing connection metadata | Clear connected Agent OS; implementation-specific selector and credential-set controls leave the normal path | Explicit secondary connection editing | Design read at the operator's window size |
| Retained target choices | Public AgentDefinitions + FleetTenants Stores | No intent for another seat's assigned credential; own selection remains available | Concurrent refusal remains visible and keeps the prior selection | Real-Store and round-trip controls |
| Credential repair | Existing main-process custody | Reasoned recovery request, without a mandatory second-token rule | Existing failure is preserved until corrected | Prompt and rejection controls; no secret in Body state |

## Acceptance Criteria

- [x] Clio has read the complete proposed Add Agent → Start journey, including ordinary, exceptional-input and failed-credential states, before implementation resumes (`MESSAGE:ae0dc9aa-c22d-486e-ba5c-79bc75c9d88d`).
- [ ] On the ordinary connected-plane path, Add Agent accepts one seat PAT and Start needs no second credential entry or required Agent Details visit.
- [ ] The normal path identifies the connected Agent OS clearly and contains no competing `This fleet`/same-URL choices or required credential-set button.
- [ ] Add names the cockpit's bound Agent OS in one quiet destination line, without a selector. The PAT field states what the token admits in one sentence; a supported two-credential case explains the distinction beside those inputs.
- [ ] Any supported case needing additional credential input explains and collects it in the same setup flow. A later credential prompt is a named recovery action, not undisclosed initial setup.
- [ ] Any retained target selector cannot emit an intent for a saved tenant credential assigned to another seat. The current assignee can retain its target; Store changes update availability; pending/rejected/accepted responses preserve existing race guards.
- [ ] The credential prompt no longer prescribes a separate identity-only PAT. Credential bytes remain in the existing main/Brain custody path.
- [ ] Retained interactive controls remain keyboard-focusable and preserve focus through pending, accepted and rejected updates. A browser witness reads `document.activeElement` after the click and update, including a retained secondary control; moving a broken control elsewhere does not satisfy the criterion.
- [ ] Start retains the observed failure and offers no complete-token repair while the typed replacement contract in neomjs/neo-agent-brain#815 is unavailable. Do not expose the existing plane-only setter as that repair. The future supported authentication-recovery copy is `The plane refused this credential. Enter a new token for this seat.` Add `expired` or `revoked` only when established; transport, identity and plane-mismatch failures retain their real diagnostics. This scope correction is Clio's `MESSAGE:36c44c60-6cbb-40c8-a79a-8673d2a3825f`.
- [ ] [L4-deferred — installed handoff needed; Residual-Owner: neomjs/neo-agent-brain#571] The next installed candidate carrying neomjs/neo-agent-brain#809 is witnessed from Add Agent through Start with one PAT entry. The witness records the actual first session's folder, loaded memory and working MCP access, or explicitly leaves the usable-seat outcome open under neomjs/neo-agent-brain#571. A green process alone cannot close that outcome.

## Post-Merge Validation

Use the installed candidate and the actual Add flow, not a pre-populated secondary credential store. Verify the ordinary path, the retained rejection without a premature repair affordance, and the unavailable saved-tenant choice. Installation is owned by #12; actual seat continuity remains owned by neomjs/neo-agent-brain#571. Neither outcome is claimed by source tests alone.

## Out of Scope

Removing the registry identity guard, replacing another tenant's stored credential, new authentication or connection modes, the instance-switcher bug (#500), moving live seats or rewriting their memory, and the separate Claude session-folder/hook repair.

## Decision Record Impact

Retains ADR 0034's main-owned credential ingress. Corrects the operator-facing requirement introduced in #411/#413. neomjs/neo-agent-brain#809 owns backend admission; this leaf owns the visible journey.

## Related

#411 / #413 · #12 · neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#809.

This is an amendment to the existing self-assigned repair leaf, not a new filing. The original live duplicate sweep, Store ownership investigation and MC prior-art records (`a9dd00ca`, `d9dd4b9c`) remain its provenance. The 2026-10-03 Clio design exchange replaces the original component-only scope. Both implementation branches remain clean at this amendment.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16
Retrieval Hint: "one PAT Add Agent Start complete journey no hidden plane credential setup"


## Timeline

- 2026-10-03T11:48:34Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-03T11:48:35Z @neo-gpt-emmy added the `bug` label
- 2026-10-03T11:48:36Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-03T11:48:36Z @neo-gpt-emmy added the `ai` label
- 2026-10-03T12:04:34Z @neo-fable-clio cross-referenced by #507
### @neo-gpt-emmy - 2026-10-03T12:06:58Z

## Implementation paused for the failed-pilot recovery

The operator escalated the unusable Mnemosyne session and stopped further speculative FM changes/migrations. This issue remains assigned to Emmy, but its branch is clean: no source changes, commit or PR has been produced. The implementation helper was interrupted.

Clio and Emmy are reconciling one Add Agent → Start journey and the actual-session acceptance on neomjs/neo-agent-brain#571 before this repair resumes. The four Neo MCP rows exist for the provisioned workspace; the resumed Claude session used the old workspace, where those rows do not exist. File-generation and process-start receipts are insufficient.

No further seat moves or runtime/config rewrites are authorized by this issue's existence. Preserve the current profile and memory copies. The design read and real-session recovery come first.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16

- 2026-10-03T12:19:34Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-03T12:26:53Z @neo-gpt-emmy changed title from **Agent setup asks once and offers only usable Agent OS targets** to **Add Agent supplies everything needed for Start**
- 2026-10-03T12:56:42Z @neo-gpt-emmy cross-referenced by #815
- 2026-10-03T14:23:00Z @neo-gpt-emmy cross-referenced by PR #515
- 2026-10-03T14:41:18Z @neo-gpt-emmy referenced in commit `16da90a` - "test(fleet): refresh one-token journey visuals (#503)"
- 2026-10-03T15:00:29Z @neo-gpt-emmy referenced in commit `096888a` - "test(fleet): check token-purpose validation spacing (#503)"
- 2026-10-03T16:55:05Z @neo-gpt-emmy added this to the **FM v1** milestone
- 2026-10-03T17:33:14Z @neo-fable cross-referenced by #351
- 2026-10-03T18:44:08Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T18:44:08Z @neo-opus-ada unassigned from @neo-gpt-emmy
### @neo-opus-ada - 2026-10-03T18:44:09Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-gpt-emmy`
**New assignees:** `neo-opus-ada`
**Reason:** Explicit author handoff from @neo-gpt-emmy (MESSAGE:3a286748-3cd7-4428-a516-3edef04781c0, 2026-10-03T18:22Z): Ada repairs PR #515's dev conflict and paired Brain pin; Emmy stays co-assignee and original author.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-03T18:44:24Z @neo-opus-ada assigned to @neo-gpt-emmy
- 2026-10-03T18:44:24Z @neo-opus-ada unassigned from @neo-opus-ada
### @neo-opus-ada - 2026-10-03T18:44:26Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-opus-ada`
**New assignees:** `neo-gpt-emmy`
**Reason:** Restoring the original author as co-assignee beside the repair owner, per the same handoff (MESSAGE:3a286748-3cd7-4428-a516-3edef04781c0).

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-10-03T18:44:50Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T18:48:50Z @neo-opus-ada referenced in commit `5d186a8` - "chore(agentos): merge dev into the one-token Add Agent branch (#503)

# Conflicts:
#	test/playwright/visual/__screenshots__/baseline-inputs.txt"
- 2026-10-03T18:48:51Z @neo-opus-ada referenced in commit `215a623` - "build(deps): pin the Brain at #818's merge commit on dev instead of its branch head, in package.json, the lock and the CI checkout (#503)"
- 2026-10-03T19:12:47Z @neo-opus-ada cross-referenced by #521
- 2026-10-03T19:25:18Z @neo-opus-ada referenced in commit `c8b8c82` - "fix(agentos): one name for the bound Agent OS on Add and on the card, a true shell-mode help line, and the detail seed's rationale (#503)

Addresses review 5402317787 on #515:
- RA-1: under shell credential ingress the help line names the next step (the
  native prompt on Add) instead of describing the removed token field; the
  shell-mode arm asserts it.
- RA-2: displayBoundAgentOs names the bound Agent OS for the Add form, the
  card's destination line and its "This fleet" chip; the chip keeps the full
  address as its hover, as a saved connection does.
- RA-3: the detail seed comment regains the owner-held selection and
  shell-owned return-verb rationale.

The restored comment put cockpit/Container.mjs over the 1000-line bar. The
three identical bound-Agent-OS bind blocks (two cockpit panes, the Accounts
panel) become one boundAgentOsBind, which takes it back to 994. The one-line
config docs on the card and the panel now state intent (non-blocking note)."
- 2026-10-03T19:30:31Z @neo-opus-ada referenced in commit `ea75678` - "test(visual): restamp the baseline inputs after the bound Agent OS naming change; the visual suite holds its goldens unchanged (#503)"
- 2026-10-03T19:46:31Z @neo-opus-ada cross-referenced by #522

