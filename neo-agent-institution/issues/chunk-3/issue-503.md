---
id: 503
title: Add Agent supplies everything needed for Start
state: OPEN
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-03T11:48:34Z'
updatedAt: '2026-10-03T12:26:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/503'
author: neo-gpt-emmy
commentsCount: 1
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
---
# Add Agent supplies everything needed for Start

## Context

The operator's 2026-10-03 Mnemosyne onboarding attempt accepted a PAT in Add Agent, then demanded a second identity-only PAT through a button in Agent Details. The operator explicitly rejected that contract. The adjacent URL choice also reverted immediately: live readback showed that its saved credential was already assigned to Sophie.

The [authority correction](https://github.com/neomjs/neo-agent-brain/issues/699#issuecomment-5968691394) records the unapproved credential burden. Brain neomjs/neo-agent-brain#809 owns admission to the connected plane using the PAT already supplied. This leaf owns a coherent Add Agent → Start experience, not just replacement wording on the credential button.

**Design status:** revised with Clio's journey requirements; awaiting her design read. Implementation remains paused. No code or installed-seat change is part of this planning update.

## The Problem

Add Agent does not communicate or collect everything that ordinary Start needs. The operator must discover implementation-specific target and credential controls in Agent Details, then encounters a second token request and a URL choice that the registry rejects. Registration therefore looks like completed setup while required input is still hidden elsewhere.

## The Architectural Reality

- The connected Fleet plane is already known. `This fleet` and a saved URL can describe the same endpoint while selecting different credential authorities; these are not interchangeable identities.
- Brain #809 supplies the existing default-plane proof-and-store path from the seat's already stored PAT. Explicit tenant credentials remain a separate authority.
- `AgentConfigComponent.createTargetChoices/onCardClick` currently offer connected tenants without checking whether another seat owns their saved credential. Brain `FleetRegistryService.configureAgent` deliberately rejects that reuse.
- The root Provider owns the public AgentDefinitions and FleetTenants Stores. `ConfigIntentRoundTrip` writes the selected definition only after an accepted response and retains its race guards.
- `harness/credentialPrompt.mjs` owns secret input in the main process. Credentials do not belong in renderer records or messages.

## The Intended Journey

1. **Add Agent collects the required input in one flow.** Retain the existing identity, harness and repository inputs. For the ordinary connected-plane case, enter the seat's PAT once. Show the connected Agent OS as a clear destination, rather than two competing badges for the same URL.
2. **The normal path has no plane selector or credential-set button.** A newly added seat does not require a visit to Agent Details to supply additional setup before Start. Connection editing is an explicit secondary operation; it must not hide a prerequisite for the normal path.
3. **Start uses what Add Agent collected.** With Brain #809 present, the ordinary connected-plane flow does not ask for another PAT. If a supported connection genuinely requires different credentials, the same Add flow must explain and collect that requirement before claiming setup is complete. Do not add a new connection mode in this leaf.
4. **A credential request at Start is recovery.** Expired, revoked or rejected credentials produce a clear repair action that names the failure. The prompt must not prescribe a second identity-only PAT or categorically forbid the existing seat PAT.
5. **Retained connection choices are usable and truthful.** A saved tenant credential assigned to another seat is not actionable. Explain why if it is shown; preserve the current assignee's choice. Refresh availability from the shared Stores. A concurrent reassignment can still be refused by Brain with a visible reason.

The failed pilot's session-folder, memory and hook findings remain under Brain #571. One-PAT admission does not certify a usable Claude session. This leaf must not imply that process readiness proves those effects.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / recovery | Evidence |
| --- | --- | --- | --- | --- |
| Add Agent → Start | Operator correction; connected plane; Brain #809 | One PAT entry in the ordinary flow; no required detour through Agent Details | Necessary exceptional input is explained in the same flow | Complete journey on the installed candidate |
| Primary configuration | Existing connection metadata | Clear connected Agent OS; implementation-specific selector and credential-set controls leave the normal path | Explicit secondary connection editing | Design read at the operator's window size |
| Retained target choices | Public AgentDefinitions + FleetTenants Stores | No intent for another seat's assigned credential; own selection remains available | Concurrent refusal remains visible and keeps the prior selection | Real-Store and round-trip controls |
| Credential repair | Existing main-process custody | Reasoned recovery request, without a mandatory second-token rule | Existing failure is preserved until corrected | Prompt and rejection controls; no secret in Body state |

## Acceptance Criteria

- [ ] Clio has read the complete proposed Add Agent → Start journey, including ordinary, exceptional-input and failed-credential states, before implementation resumes.
- [ ] On the ordinary connected-plane path, Add Agent accepts one seat PAT and Start needs no second credential entry or required Agent Details visit.
- [ ] The normal path identifies the connected Agent OS clearly and contains no competing `This fleet`/same-URL choices or required credential-set button.
- [ ] Any supported case needing additional credential input explains and collects it in the same setup flow. A later credential prompt is a named recovery action, not undisclosed initial setup.
- [ ] Any retained target selector cannot emit an intent for a saved tenant credential assigned to another seat. The current assignee can retain its target; Store changes update availability; pending/rejected/accepted responses preserve existing race guards.
- [ ] The credential prompt no longer prescribes a separate identity-only PAT. Credential bytes remain in the existing main/Brain custody path.
- [ ] The next installed candidate carrying Brain #809 is witnessed from Add Agent through Start with one PAT entry. The witness records the actual first session's folder, loaded memory and working MCP access, or explicitly leaves the usable-seat outcome open under #571. A green process alone cannot close that outcome.

## Post-Merge Validation

Use the installed candidate and the actual Add flow, not a pre-populated secondary credential store. Verify the ordinary path, a rejected credential's recovery path, and the unavailable saved-tenant choice. Installation is owned by #12; actual seat continuity remains owned by neomjs/neo-agent-brain#571. Neither outcome is claimed by source tests alone.

## Out of Scope

Removing the registry identity guard, replacing another tenant's stored credential, new authentication or connection modes, the instance-switcher bug (#500), moving live seats or rewriting their memory, and the separate Claude session-folder/hook repair.

## Decision Record Impact

Retains ADR 0034's main-owned credential ingress. Corrects the operator-facing requirement introduced in #411/#413. Brain #809 owns backend admission; this leaf owns the visible journey.

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

