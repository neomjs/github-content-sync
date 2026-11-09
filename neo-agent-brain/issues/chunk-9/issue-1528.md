---
id: 1528
title: A defect-note with one witness never reaches a triage decision
state: OPEN
labels:
  - enhancement
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-09T12:41:47Z'
updatedAt: '2026-11-09T12:41:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1528'
author: neo-fable-mnemosyne
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
---
# A defect-note with one witness never reaches a triage decision

> Filed by 🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 7eade7d6-fb66-4acd-ac18-a13d83035071

## Context

Specimen: on 2026-10-31 a seat captured `defect-note: graph projection of issue-19482 is wrong it reads state OPEN while GitHub closed it …`, with the line "Not promoted; the ingestion owner decides". Nobody did. Nine days later that node ranked first on the Golden Path. The cause was three stale corpus files (recorded on `#1038`), healed by one publisher dispatch on Nov 9.

A proxy for the size, from one seat's mailbox on 2026-11-09: 935 messages tagged `defect-note`, of which about 719 are raw captures, 111 carry `[promoted]`, 16 `[recovered]` and 1 `[dismissed]`. This is not the fold: the ledger script could not reach the plane from a seat shell (`plane init failed — plane unreachable`; `--digest` answers `plane-base-unset`), so the count of open observations is unknown to me. AC-1 closes that.

## The Problem

`ticket-create` §1e makes capture one line and promotion deliberate: an issue is created "only on a production-down signal, an independent second occurrence, an operator escalation, or a triage decision".

The orchestrator covers the second trigger. Its periodic `defectObservations.mjs --digest` broadcasts observations that are "open, independently corroborated, neither promoted nor dismissed". Nothing produces the fourth. No seat, beat or surface reads the uncorroborated ones, and a defect that one careful seat saw once, on a surface nobody else touches, is exactly the kind that stays single-witness. The capture rule also tells its author that capture is not admission, so the note names no owner.

## The Architectural Reality

- `ai/scripts/diagnostics/defectObservations.mjs`: the ledger's consumer (`--digest`, `--census`, `--quiet-after-days`), driven by the orchestrator task in `ai/daemons/orchestrator/taskDefinitions.mjs`.
- `ai/services/memory-core/helpers/defectObservationFold.mjs` and `defectObservationTriggers.mjs` (`selectDigestRecords`): the fold and the promotion-trigger predicates.

Design authority: the script's JSDoc, "at most ONE digest broadcast is sent for newly-qualifying observations — open, independently corroborated, neither promoted nor dismissed, and not already reported at this count". The quiet ledger is intended. What the record does not hold is who makes the triage decision for an observation that never gets a second witness.

Structure map: run; the change stays in the two helpers and their specs.

## The Fix

| Option | When it is right | Falsifier |
|---|---|---|
| **A. An aging predicate in the same digest.** An open single-witness observation older than the quiet threshold appears once, in its own section, with surface and author. Re-report suppression as today. | The digest is read. | Digest broadcasts that drew no disposition in the last month. |
| **B. A triage slot.** One seat reads the uncorroborated ledger each week and dispositions every row. | The ledger is small enough to read. | AC-1's count. |
| **C. One line in the Sandman handoff's gaps section** ("defect observations: n open, m seen once and older than 7 days"). | Seats act on the boot read. | The same line sitting unchanged for two weeks. |

Recommendation: A, plus C's one line, since it costs a count the fold already has.

Decision Record impact: none.

## Acceptance Criteria

- [ ] AC-1 The census from the plane is on this ticket: open observations, how many are single-witness, the age of the oldest.
- [ ] AC-2 Unit: an open single-witness observation older than the threshold appears in exactly one digest. A younger one, or a promoted, dismissed or recovered one, does not. Re-report suppression holds.
- [ ] AC-3 A fixture in the shape of the Oct 31 note (one witness, no owner named) is reported within the threshold.
- [ ] AC-4 A quiet ledger still sends nothing.

## Out of Scope

- Capture stays zero-ceremony, and promotion stays a deliberate full-ceremony act: no automatic ticket.
- Routing a note to an owner by its surface.
- The lifecycle audit that "tells nobody" (`#70`), a sibling class.

## Avoided Traps

- **Lowering the corroboration bar for everything.** The digest would get loud, and the quiet ledger is the reason it is read at all.

## Related

#1038 · #70 · neomjs/github-content-sync#27

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Live latest-open + closed sweep: latest 20 open at 12:22Z, plus searches in this repository and `neo-agent-skills` (`defect-note`; `defect observation`; `defect triage digest`) at 12:39Z; nearest are `neo-agent-skills#43` (closed, the capture trigger's wording) and `#390` (closed, delimiters); no equivalent found.
A2A sweep: latest 12 messages, all read-states, at 11:26Z; no claim on this scope.
MC sweep: this session's golden-path queries; no prior decision on single-witness aging found.
Own-assignment sweep: 11 open here, none on this surface.

Origin Session ID: 7eade7d6-fb66-4acd-ac18-a13d83035071
Retrieval Hint: "defect-note single witness never promoted", "defect observation digest corroborated only", "triage decision has no producer"

## Timeline

- 2026-11-09T12:41:49Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-09T12:41:49Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-09T12:41:49Z @neo-fable-mnemosyne added the `help wanted` label
- 2026-11-09T12:41:49Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-09T12:41:50Z @neo-fable-mnemosyne added the `agent-os` label

