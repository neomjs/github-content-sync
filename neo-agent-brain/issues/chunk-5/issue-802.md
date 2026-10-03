---
id: 802
title: ADR 0041 records the run's witness section and its reconciliation arm
state: CLOSED
labels:
  - documentation
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-03T08:41:47Z'
updatedAt: '2026-10-03T10:36:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/802'
author: neo-fable-clio
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-03T10:36:43Z'
---
# ADR 0041 records the run's witness section and its reconciliation arm

The ADR-update half of #782, split out per ADR 0005 §6.5 ("file a separate ADR-update PR first; do not amend the implementation PR to also touch the ADR" — the same split neomjs/neo-agent-brain#788 → PR #789 made for the host-file settle sentence). Sub of neomjs/neo-agent-institution#351.

## The amendment

`learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`:

- **§2 gains item 9** (dated 2026-10-03): the record's `verification` section — what the served plane answered when THIS run's witness memory was written (`attempt` durable before dispatch, `memory`, `readback`, `recall`; the host's attempt metadata and observation stamps beside the memory identity, timestamp, session and recall hit returned by the plane), written by the one writer beside the `verify` receipt, retired with the receipts on rebinding; historical by construction — `validation` stays a fresh observation that reads no receipt, and `done` turns `ok` only in an evaluation whose `served-plane` and `validation` are fresh and `ok` beside the witness; one dispatched write per attempt, a lost acknowledgement adopted from a positive read or left `reconcile-required`, never replayed; a second write only as a new attempt the operator consents to explicitly.
- **§3 gains a third witness arm**: write the witness → lose its acknowledgement → read an empty or unavailable recency page → resume twice; expected one attempt, no second row, `reconcile-required` until a read carries the attempt's marker; and a record holding a complete witness against a served plane that no longer matches reads `done` not ok with the witnessed timestamp in its reason.

The readiness authority of §2.3 / §2.5 / §4 is unchanged; the one-writer rule holds; the §3 arm is the test evidence the implementing PR (#796) carries.

## Acceptance Criteria

- AC-1: §2.9 and the §3 arm land on `dev` ahead of the implementation PR (#796), which then depends on this record and no longer touches the ADR.
- AC-2: the text names the contract the implementation's specs pin by name: durable attempt before dispatch, adopt-or-reconcile, consented new attempt, `done` gated on fresh `served-plane` + `validation`.

## Out of Scope

The implementation (#782 / PR #796). The host-file settle sentence (#788 / PR #789, the neighbouring amendment of §3 — whichever lands second rebases the paragraph).

## Related

#782 · PR #796 · #788 / PR #789 · ADR 0005 §6.5 · neomjs/neo-agent-institution#351

Live latest-open sweep: the latest 20 open Brain issues read at 2026-10-03T08:42Z; #788 is the only open ADR 0041 amendment and covers a different sentence. A2A: Sophie's #796 gate (08:35Z) asked for exactly this split.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "ADR 0041 §2.9 verification section witness arm split ADR-update PR first"

## Timeline

- 2026-10-03T08:41:47Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-03T08:41:49Z @neo-fable-clio added the `documentation` label
- 2026-10-03T08:41:49Z @neo-fable-clio added the `ai` label
- 2026-10-03T08:41:50Z @neo-fable-clio added the `architecture` label
- 2026-10-03T08:41:50Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T08:41:59Z @neo-fable-clio added parent issue #351
- 2026-10-03T08:42:36Z @neo-fable-clio cross-referenced by PR #803
- 2026-10-03T08:43:09Z @neo-fable-clio cross-referenced by PR #796
- 2026-10-03T08:46:05Z @neo-fable-clio referenced in commit `bbc31b3` - "docs(adr): ADR 0041 §2.9 records the run's verification section and §3 its reconciliation arm (#802)"
- 2026-10-03T08:57:43Z @neo-fable-clio cross-referenced by #782
- 2026-10-03T09:00:09Z @neo-fable-clio referenced in commit `939b47b` - "docs(adr): §2.9 tells the host's attempt metadata from the plane's returned values (#802)"
- 2026-10-03T10:36:43Z @tobiu referenced in commit `8fec9a4` - "docs(adr): ADR 0041 §2.9 records the run's verification section and §3 its reconciliation arm (#802) (#803)

* docs(adr): ADR 0041 §2.9 records the run's verification section and §3 its reconciliation arm (#802)

* docs(adr): §2.9 tells the host's attempt metadata from the plane's returned values (#802)"
- 2026-10-03T10:36:44Z @tobiu closed this issue

