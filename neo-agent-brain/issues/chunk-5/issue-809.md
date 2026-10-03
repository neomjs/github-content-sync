---
id: 809
title: Start uses the existing seat PAT for the connected Agent OS
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-10-03T11:41:49Z'
updatedAt: '2026-10-03T12:06:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/809'
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
# Start uses the existing seat PAT for the connected Agent OS

## Context

The operator's Mnemosyne pilot on 2026-10-03 accepted the seat's PAT through Add Agent, then refused Start because a separate plane-credential slot was empty. The UI instructed creation of a second identity-only PAT. The operator explicitly rejected that added requirement. The authority correction and review accountability are recorded in [5968691394](https://github.com/neomjs/neo-agent-brain/issues/699#issuecomment-5968691394).

## The Problem

The normal connected-Fleet journey already possesses the seat's credential, but treats an empty secondary store as missing operator input. A separate credential purpose does not establish a need for a second human-created token. The installed setter checks identity and served-plane binding; it does not require distinct credential bytes or enforce repository-scope restrictions. The pilot subsequently started through ordinary Fleet Start with the same existing PAT bound for the plane.

## The Architectural Reality

- `FleetRegistryService.resolveCredential(id)` is the Brain-internal accessor for the PAT supplied once at definition; it is never a wire getter.
- `resolveSeatPlaneTarget` distinguishes the Fleet's attached plane from an explicitly selected tenant.
- `startAgentProvisioned` currently rejects a default-plane seat when `resolveSeatPlaneCredential` returns null.
- `FleetTenantService.storeSeatPlaneCredential` already proves the seat identity and matching MC/KB plane tuple before encrypted persistence. `probeSeatPlaneCredential` revalidates the stored tuple before use.
- `armFleetSeatWake` consumes that verified binding. The explicit tenant branch has a separate credential authority and must retain it.

## The Fix

For a default-plane seat whose binding is missing, use the existing registry PAT internally to establish the binding through the current proof-and-store path before provisioning/spawn. Retain existing explicit bindings, identity verification, served-plane revalidation and controlled refusal. Do not send a checkout credential to an arbitrary selected tenant or expose it to the renderer. Update the runbook and method contracts that prescribe a second identity-only PAT as a universal requirement. The paired Institution repair owns the misleading prompt and unavailable target choice.

## Contract Ledger

| Surface | Authority | Behavior | Refusal / fallback | Evidence |
| --- | --- | --- | --- | --- |
| Default-plane Start | Current operator correction; `startAgentProvisioned` | Missing binding is established from the seat's already stored PAT through existing proofs | A rejected identity, unavailable plane or failed persistence prevents spawn; no private MC/KB fallback | Temporary-store integration control |
| Bound-plane reuse | Existing `probeSeatPlaneCredential` contract | Continue proving the stored identity and plane id/dataRoot | Changed plane remains refused; no implicit rebinding | Existing drift controls |
| Explicit tenant | Existing tenant authority | Continue using the selected tenant's credential | No registry-PAT substitution | Negative control |
| Secret custody | Existing registry/tenant stores | Credential stays Brain-side; encrypted persistence only after proof | No secret in results/logs; corrupt stores preserved | Store and redaction controls |

## Acceptance Criteria

- [ ] A newly defined repo-bearing seat with only its normal PAT and the default connected-plane target can pass the ordinary Start flow without `setPlaneCredential` or another credential entry.
- [ ] The first binding uses the same credential and the existing expected-seat / matching MC-and-KB plane proofs; wrong identity, unreachable plane and persistence failure do not clone or spawn.
- [ ] Existing binding and changed-plane refusal controls remain valid; the explicit tenant branch never silently receives the registry PAT.
- [ ] Repeated starts do not create another identity or require another human-created token, and wake arming uses the admitted binding.
- [ ] Runbook and JSDoc no longer prescribe a separate identity-only PAT for every default-plane seat.

## Post-Merge Validation

The next newly added peer in the installed Fleet completes Add Agent → Start with one PAT entry and performs a peer-visible memory operation. Residual-Owner: #571.

## Out of Scope

New authentication protocols, a token minter, changing other tenant credentials, the copied app-profile/memory migration, and the Institution selector rendering.

## Avoided Traps

A copied token alone is not admission: the existing subject and served-plane proofs stay mandatory. A new operator token or hidden second entry is not the repair.

## Decision Record impact

Aligned with ADR 0019's config ownership and ADR 0041's plane-binding proof. Corrects the agent-selected credential burden in `#699`; no new AiConfig leaf or transport mode.

## Related

#571 · #699 / #728 · neomjs/neo-agent-institution#411 / neomjs/neo-agent-institution#413 · #797 / #806 (same start file, separate memory-import block).

Live latest-open/A2A sweep: latest 20 Brain issues and 30 current messages checked immediately before filing; no equivalent or overlapping credential fix. Vega confirms no overlap; Ada flags only #806's separate block. MC rationale: memories `d9dd4b9c` and `df7e7229` retain the original Tier-2 decision; current operator correction supersedes its extra-token requirement. Own-assignment sweep: #426/#306/#48 are unrelated. Structure map completed; existing owner is `ai/services/fleet`, no new directory.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16
Retrieval Hint: "Add Agent accepted PAT Start missing plane credential one-PAT operator correction"

## Timeline

- 2026-10-03T11:41:49Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-10-03T11:41:50Z @neo-gpt-emmy added the `bug` label
- 2026-10-03T11:41:50Z @neo-gpt-emmy added the `ai` label
- 2026-10-03T11:41:51Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-03T11:48:35Z @neo-gpt-emmy cross-referenced by #503
### @neo-gpt-emmy - 2026-10-03T12:06:57Z

## Implementation paused for the failed-pilot recovery

The operator escalated the unusable Mnemosyne session and stopped further speculative FM changes/migrations. This issue remains assigned to Emmy, but its branch is clean: no source changes, commit or PR has been produced. The implementation helper was interrupted.

Clio and Emmy are reconciling one Add Agent → Start journey and the actual-session acceptance on #571 before this repair resumes. The four Neo MCP rows exist for the provisioned workspace; the resumed Claude session used the old workspace, where those rows do not exist. File-generation and process-start receipts are insufficient.

No further seat moves or runtime/config rewrites are authorized by this issue's existence. Preserve the current profile and memory copies. The design read and real-session recovery come first.

Origin Session ID: b42bdfd4-f066-4093-8602-c02f56bd9f16

- 2026-10-03T12:19:34Z @neo-gpt-emmy cross-referenced by #571

