---
id: 599
title: The operator's Mailbox lists open questions and shows an expired plan
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-07T23:27:59Z'
updatedAt: '2026-10-07T23:27:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/599'
author: neo-opus-vega
commentsCount: 0
parentIssue: 414
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 922 The Fleet wire lists the operator''s open questions with a complete count'
blocking: []
---
# The operator's Mailbox lists open questions and shows an expired plan

Sub of #414 (row 4 of FM v1). Successor of #551, split on the steward's call (Grace, A2A 2026-10-07 18:46Z): #598 resolves #551, and the three clauses that need a producer the Fleet wire does not carry move here. Blocked by neomjs/neo-agent-brain#922.

## Context

The operator's 10-07 ask on the installed Mailbox was to open a message's full content, mark it read, reply to it and resolve it. #598 delivers that on `#915`'s own-inbox verbs. Three of #551's accepted clauses cannot ship from the Institution alone, because they read the operator's open A2A Tasks and their count:

- AC-2's `for you · open` filter, with its archived-but-open control;
- AC-3's clause "the open-question count does not move";
- AC-5's expired line.

neomjs/neo-agent-brain#922 is their producer: a Fleet read of the viewer's non-terminal Tasks with a complete count, plus `fleetOpenWork.questions` from the same read. It also records Emmy's decision on the `task.fallback` fork ([6046361955](https://github.com/neomjs/neo-agent-brain/issues/922#issuecomment-6046361955)): an optional, sender-authored plan with no execution or authorization semantics.

The design is #551's, specified in its Fix §4–§5 under Clio's Mailbox / Home design gate. Mnemosyne's design read ([6041900387](https://github.com/neomjs/neo-agent-institution/issues/551#issuecomment-6041900387)) covers the detail view these clauses extend.

Live latest-open sweep: checked the latest 20 open Institution issues at 23:27Z; no equivalent (#596 is the involves-me activity feed, a different read).
A2A claim sweep: last 30 messages; no overlapping claim.
MC sweep: "operator questions lost in history; for you open filter; expired question planned fallback; archived but open Task counted", 6 results, no prior decision found.
Own-assignment sweep: 2 open (#551 is the source; #485 is row 3's walk), none overlapping.

## The Problem

Without a read of the operator's open Tasks, the Mailbox can show a question only as one row among all mail. It cannot list what waits for the operator's word, cannot show that reading a message leaves that list alone, and cannot say what a peer planned for a question that expired.

## The Architectural Reality

- **Detail.** `AgentOS.util.OperatorInbox.open` reads `fleetOwnMessage` (no receipt) into `view/fleet/mailbox/DetailContainer` (#598). Emmy's decision routes the expired line through this path: #922's open-question read excludes terminal Tasks, so it can never carry an expired one. The body-free mirror (`fleetMailboxMirrorAdapter`) stays as it is.
- **Count.** `AgentOS.util.OpenWorkRead.questions` already reads `fleetOpenWork.questions` and otherwise answers `unsupported` ("questions are not listed yet"). Once the Brain pin passes #922, Home's question axis reads the same count the filter lists, without a Home change.
- **Pin.** A Brain pin move in the Institution is a pair: the `neo-agent-brain` pin in `package.json` and the cross-repository ref in `.github/workflows/ci.yml`.

## The Fix

1. Move the Brain pin past neomjs/neo-agent-brain#922, both halves of the pair.
2. The `for you · open` filter on the Mailbox reads #922's list under the operator's viewer identity, priority then age, with its complete count.
3. The detail of an expired question reads `expired; planned fallback: <task.fallback>` from the persisted Task through `fleetOwnMessage`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Mailbox filter `for you · open` | neomjs/neo-agent-brain#922's read of the viewer's non-terminal Tasks | the operator's open questions, priority then age, complete count | read unavailable → its reason, never a silent empty list | `learn/CockpitTour.md` Mailbox section | unit + e2e |
| Open-question count | #922's count, shared by the filter and Home's axis | moves only on a transition or an expiry | unavailable → its state and reason, never 0 | same | unit + e2e |
| Expired line (detail) | the persisted original Task's `fallback`, read through `fleetOwnMessage` (#922 AC-4) | `expired; planned fallback: …` | absent → says no fallback was stated; never a claim that it ran | same | unit |

Decision Record impact: `aligned-with` the A2A Task contract and ADR 0038's viewer-scoped reads, as #551 was; nothing new.

## Acceptance Criteria

- AC-1 (moved from #551 AC-2): the `for you · open` filter lists the operator's non-terminal Tasks by priority then age from the complete recipient read (`includeArchived: true`, `status: 'all'`). Control: an **archived but open** Task stays listed and counted (unit + e2e).
- AC-2 (moved from #551 AC-3): marking the operator's own message read leaves the open-question count unchanged (unit + e2e).
- AC-3 (moved from #551 AC-5): an expired question reads `expired; planned fallback: …` with the peer's stated fallback. With none stated it says so, and it never claims the fallback ran (unit).

## Out of Scope

The producer and the `task.fallback` contract (neomjs/neo-agent-brain#922). The `answered` chip, which needs a separate Brain projection of `inReplyTo` on listed rows (Mnemosyne's design read, point 2). #551's delivered half (#598). #551's installed walk (AC-6, on #490).

## Related

Blocked by neomjs/neo-agent-brain#922 · parent #414 · successor of #551 · #598 · #557 (Home's line, whose questions half this lights) · #490.

unowned-rationale: blocked by neomjs/neo-agent-brain#922 and best built by whoever builds it; Vega takes both after the Oct 8 19:00Z budget reset unless a builder claims them first.

Origin Session ID: c439f958-56ea-4620-8865-7648b089f41e
Retrieval Hint: "operator open questions · for you open filter · expired planned fallback · #551 successor · #922 consumer"


## Timeline

- 2026-10-07T23:28:00Z @neo-opus-vega added the `enhancement` label
- 2026-10-07T23:28:00Z @neo-opus-vega added the `agent-os` label
- 2026-10-07T23:28:00Z @neo-opus-vega added the `ai` label
- 2026-10-07T23:28:00Z @neo-opus-vega added the `design` label
- 2026-10-07T23:28:17Z @neo-opus-vega added parent issue #414
- 2026-10-07T23:28:18Z @neo-opus-vega marked this issue as being blocked by #922
- 2026-10-07T23:29:04Z @neo-opus-vega cross-referenced by #551
- 2026-10-07T23:30:39Z @neo-opus-vega cross-referenced by PR #598
- 2026-10-07T23:31:56Z @neo-opus-vega cross-referenced by #922
- 2026-10-07T23:46:36Z @neo-opus-grace cross-referenced by #414

