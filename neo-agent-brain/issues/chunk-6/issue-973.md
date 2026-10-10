---
id: 973
title: 'The public proof reasons are a contract export, so consumers can gate on them'
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T16:41:15Z'
updatedAt: '2026-10-10T17:22:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/973'
author: neo-fable-clio
commentsCount: 0
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
closedAt: '2026-10-10T17:22:56Z'
---
# The public proof reasons are a contract export, so consumers can gate on them

## Context

#964 / PR #966 put the launch issuer's bounded proof diagnostics into the admission audit (`recent[].reason` for `proof-unavailable`), admitted by `isPublicProofReason` — a `Set` of producer-owned wordings plus the `plane MCP readiness failed (<status>)` pattern that lives in `ai/services/fleet/mcpLaunchAdmission.mjs:49–65`, a Brain-internal service module. The consumer that must print those words — the seat card in the Institution (#655 / PR #660) — imports only the Fleet contract (`src/fleet/contract/index.mjs`), so it cannot gate on the vocabulary: Sophie's #660 falsifier (2026-10-10 16:37Z) showed an unknown reason (`UNRECOGNIZED_PROBE_SENTINEL`) printed on the card, against #655's AC-1 ("an entry whose reason is outside the closed vocabulary prints no reason"). Mirroring the list in the Institution would drift the day the producer adds a wording.

Design authority: #964's contract ledger row ("`reason` … the producer's allowlisted diagnostic"; the allowlist is the contract's to publish) and #655's AC-1.

## The Fix

Lift the vocabulary into the contract and let the service import it:

- `src/fleet/contract/launchAdmission.mjs`: `export const LAUNCH_ADMISSION_PROOF_REASONS` (the frozen list of wordings, exactly today's `PROOF_REASONS`) and `export function isLaunchAdmissionProofReason(reason)` (the list plus the `plane MCP readiness failed (<status>)` pattern — the same test as today's `isPublicProofReason`).
- `ai/services/fleet/mcpLaunchAdmission.mjs`: drop the local `Set` and `isPublicProofReason`; its three callers (`McpLaunchAdmissionService.mjs:491,542`, `fleetMcpLauncher.mjs:175`) import `isLaunchAdmissionProofReason` from the contract directly — one name, one home, no alias. (Disposition 2026-10-10 17:1xZ, Sophie's ledger alignment on PR #975: the first wording kept `isPublicProofReason` as a re-export so the callers stayed untouched; the direct import is the accepted shape.)
- A contract unit spec beside `launchAuthority.spec.mjs`: the list is frozen and non-empty, each wording passes, the readiness pattern passes for 1xx–5xx and fails otherwise, an arbitrary string fails.

No wire change, no new wording, no behaviour change: the same words, exported where consumers can read them.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `LAUNCH_ADMISSION_PROOF_REASONS` + `isLaunchAdmissionProofReason` (`src/fleet/contract/launchAdmission.mjs`, new) | #964 | the issuer's public proof wordings, frozen; the matcher the issuer and its consumers share | — | contract JSDoc | unit |
| `isPublicProofReason` (`ai/services/fleet/mcpLaunchAdmission.mjs`) | #966 | removed; the three callers import the contract matcher directly (disposition 2026-10-10: no re-export alias) | — | JSDoc | the existing admission, launcher and lifecycle specs |

Decision Record impact: `none`.

## Acceptance Criteria

- [ ] AC-1 — unit: the contract list is frozen, non-empty and equal to today's wordings; the matcher admits every listed wording and `plane MCP readiness failed (503)`, refuses `plane MCP readiness failed (999)` and an arbitrary string.
- [ ] AC-2 — the three callers still pass their existing specs importing the contract name; the service holds no second list and no alias (`isPublicProofReason` has no in-tree reference).
- [ ] AC-3 — the Institution's card (PR #660) gates its diagnostic on the contract matcher after `resolve-org-dev` picks this up (consumer evidence on that PR).

## Out of Scope

New wordings; the retry budget or any refusal code; the Institution's words themselves (#655).

## Related

#964 / #966 (the vocabulary's origin), Institution #655 / PR #660 (the consumer), #477 (the words' parent).

Sweeps: live latest-20 open Brain issues read 2026-10-10 16:39:44Z — no equivalent; A2A last 30 at 16:4xZ — Vega on #968/#971, Grace on the Dependabot callers, Sophie reviewing #660; none on the contract; Memory Core — #966's trail kept the list service-local, nothing decided against a contract export; own assignments (#850 #50 #51 #53) — none on this surface; structure map: `src/fleet/contract` (sibling exports) + `ai/services/fleet`, no new file.

Retrieval Hint: "LAUNCH_ADMISSION_PROOF_REASONS contract export isPublicProofReason consumer gate"

Origin Session ID: 173797a7-0c19-4372-a7e2-5cab6f65f46b

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 173797a7-0c19-4372-a7e2-5cab6f65f46b


## Timeline

- 2026-10-10T16:41:15Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T16:41:16Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T16:41:16Z @neo-fable-clio added the `ai` label
- 2026-10-10T16:41:17Z @neo-fable-clio added the `architecture` label
- 2026-10-10T16:41:17Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T17:02:18Z @neo-fable-clio cross-referenced by PR #975
- 2026-10-10T17:22:56Z @tobiu referenced in commit `f61bba4` - "feat(fleet): the public proof reasons are a contract export (#973) (#975)

The issuer's bounded diagnostics for a proof the plane did not answer lived as a private Set beside the admission wire, where the Institution's seat card could not gate on them and printed any string. The contract now exports the frozen list and the matcher (the list plus the plane's readiness-status pattern); the issuer and the launcher import the contract name directly, so one name has one home and the service holds no second list. No wire change, no new wording."
- 2026-10-10T17:22:56Z @tobiu closed this issue
- 2026-10-10T17:30:22Z @neo-opus-vega cross-referenced by #571

