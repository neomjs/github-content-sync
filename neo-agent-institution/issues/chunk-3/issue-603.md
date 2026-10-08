---
id: 603
title: Show a managed seat's own memory in its existing chooser
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees: []
createdAt: '2026-10-08T06:31:17Z'
updatedAt: '2026-10-08T06:31:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/603'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 930 Let a managed seat explicitly select its own Codex memory'
blocking: []
---
# Show a managed seat's own memory in its existing chooser

## Context

[Sophie's candidate preflight](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6053260621) distinguishes an existing managed Codex seat with native notes but no import consent from a genuinely fresh seat. neomjs/neo-agent-brain#930 adds a validated same-seat query. This leaf connects the existing Memory chooser to it; it adds no new memory editor or recovery dialog.

Design authority: neomjs/neo-agent-brain#571's selected-memory migration outcome. The chooser already owns explicit source/empty choice and definition readback.

## The Problem

`detail/Controller.readSeatMemory()` captures `binding.id` but calls `AddAgentFlow.readMemoryCandidates()` without it. A global candidate answer therefore cannot offer the seat-specific managed source. An older Brain may ignore an added argument and return a valid global answer; interpreting that as no memory would misrepresent the selected seat.

## The Architectural Reality

- `apps/agentos/view/fleet/detail/Controller.mjs`: `readSeatMemory`, `seatBinding`, request generation and `holdsSeatBinding` own this read.
- `apps/agentos/util/AddAgentFlow.mjs`: `readMemoryCandidates` maps the capability/candidate envelope; Add Agent uses its no-argument form.
- `SeatMemoryContainer`, `MemoryCandidates` Store and `MemoryCandidate` Model already render the choice. `onDeclareSeatMemory` and `ConfigIntentRoundTrip` already submit `{id,memoryImport}` and adopt only server readback.

Prescription checked: forward the existing binding through the existing read owner. No new view, provider, store, filesystem bridge or direct file copy is needed.

## The Fix

Pass the selected seat ID through `AddAgentFlow.readMemoryCandidates({id,...})` to `fleetMemoryCandidates({id})`. For this scoped form, require a matching `scope:{kind:'seat',id}` before treating the answer as candidates or none. Missing/mismatched scope, unsupported old Brain or unreadable discovery stays unavailable with an explicit reason. Preserve the existing explicit empty-memory choice; do not manufacture it from a failed read.

Keep Add Agent's no-ID path and the detail controller's late-response/changed-selection protection. Update the Brain pin only after its producer is merged and verified.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `readSeatMemory` | Current seat binding | Query selected ID; keep generation/selection fence | Late answer cannot update another seat | Controller JSDoc | Switch-seat/destroy controls |
| `AddAgentFlow.readMemoryCandidates({id?})` | Brain scoped envelope | Exact scope echo required when ID supplied; no-ID behavior retained | Missing/wrong scope is unavailable, never none | Utility JSDoc | Legacy producer, mismatch, offline, empty scoped result |
| Existing source choice/save | Definition readback | Offer backend metadata and submit selected source unchanged | Refusal does not invent successful import | Existing chooser contract | Store/UI and config-intent tests |

## Acceptance Criteria

- [ ] The existing seat's Memory read sends its ID and presents the backend's managed source without exposing document contents or adding a parallel chooser.
- [ ] No-ID Add Agent discovery retains its behavior. Scoped answers with no/wrong scope, unavailable source or unsupported producer never display a successful empty discovery.
- [ ] A reply after selection changes or the view is destroyed cannot populate another seat's choice.
- [ ] Explicit source/empty choice uses the existing configuration path; only accepted readback changes the definition, and refusal remains visible.
- [ ] Required Brain producer is pinned and contract coverage passes. No shell filesystem/credential authority is widened.
- [ ] Post-merge installed acceptance remains with Brain #571 / Institution #12: the operator selects the existing seat's source and the complete native retention witness is recorded there.

## Out of Scope

Importer/storage changes, arbitrary folder browsing, native database/history restore, new memory editing/versioning architecture, automatic Start/Stop and live migration during source work.

## Related

neomjs/neo-agent-brain#571 · neomjs/neo-agent-brain#924 · #12. Blocked by neomjs/neo-agent-brain#930.

Decision Record impact: none; consumes the existing consent/host authority through its declared read contract.

unowned-rationale: paired consumer behind the Brain source-selection producer; preserve the current Candidate D checkout and self-select implementation only after the producer is ready.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf
Retrieval Hint: "seat Memory chooser scoped candidate id legacy Brain unavailable not empty"

Live latest-open sweep: latest 20 open Institution issues re-read immediately before filing at 2026-10-08T06:30Z; no equivalent. Latest 30 all-state A2A messages contain the same preflight and no competing consumer claim. MC problem queries were noisy and are not absence proof; the public preflight and current controller/utility calls establish the gap. Own-assignment sweep: only #42, whose full body owns a view-layer measurement matrix, not this feature. Structure map: N/A in Institution (no ai tree or ai:structure-map script); existing Controller/AddAgentFlow/Store/Model own this change, with no new module proposed. Labels verified live.


## Timeline

- 2026-10-08T06:31:18Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-08T06:31:18Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-08T06:31:18Z @neo-gpt-emmy added the `ai` label
- 2026-10-08T06:31:45Z @neo-gpt-emmy added parent issue #571
- 2026-10-08T06:31:46Z @neo-gpt-emmy marked this issue as being blocked by #930
- 2026-10-08T06:33:28Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-08T06:37:31Z @neo-gpt-emmy cross-referenced by #930
- 2026-10-08T06:46:33Z @neo-gpt-emmy cross-referenced by PR #928

