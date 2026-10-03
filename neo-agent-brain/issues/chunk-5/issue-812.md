---
id: 812
title: 'ADR 0041 §2.6 and §2.7: a receipt proves the input it recorded'
state: OPEN
labels:
  - documentation
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-03T12:46:16Z'
updatedAt: '2026-10-03T12:46:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/812'
author: neo-fable-clio
commentsCount: 0
parentIssue: 810
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
# ADR 0041 §2.6 and §2.7: a receipt proves the input it recorded

## Context

The authority half of #810 (a consent changed after an accepted effect re-applies it as a new input). ADR 0005 §6.5 orders an ADR amendment ahead of its implementation in a separate PR, and `check-pr-body` admits one standalone `Resolves #N` per PR — so the docs PR needs its own leaf, as #788 was for #786 and #802 for #803. Planner-filed under the filing freeze on @neo-fable's proposal (12:42Z); she drafted the hunks and builds the PR.

## The Problem

ADR 0041 (`learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md`) says an accepted effect is never re-run because a renderer resumed (§2.6) and that a change of target or recipe version retires every proof (§2.7). It does not say what a receipt is proof OF, so a consent change after acceptance has no rule — and the implementation kept receipts as proof for intents the operator had abandoned (#810's measured four-step probe).

## The Fix (three hunks, one PR, docs only)

1. **§2 item 6**, after "never re-run because a renderer resumed.": *"Its receipt proves the input it recorded — the content it wrote, the set it completed, the composition it started from — and that input only: when the consents render a different input, the effect is `pending` again and its next run is a new application for the new input, never a replay of the accepted one (amended 2026-10-03, neomjs/neo-agent-brain#812)."*
2. **§2 item 7**, after "they stay history.": *"A change of consent retires nothing wholesale: each receipt stays proof for the input it recorded, so only the effects whose input the change touches read `pending` again (amended 2026-10-03, neomjs/neo-agent-brain#812)."*
3. **Status row**: *"amended 2026-10-03 by neomjs/neo-agent-brain#812: §2.6 and §2.7 key a receipt to the input it recorded (one sentence each)"*.
4. **§3 witness companion** (yes — the ADR's witness discipline wants it): one sentence after the existing host-file sentence: *"A third arm: accept every effect → change one consent → the effects whose input the change touches read `pending` with the earlier-input reason, the untouched ones stay `ok`, and the next run applies only the touched ones as new inputs."*

The ticket numbers in the hunks read `#812` — this leaf's own number — if the tracker assigns it; otherwise the PR replaces them with the real one.

## Acceptance Criteria

- [ ] AC-1 The four hunks land in ADR 0041 as docs only (zero-delta change class), merged ahead of #810's implementation PR; #810's PR body names this leaf.
- [ ] AC-2 The sentences name no mechanism (no field names, no file names) — authority, not implementation.
- [ ] AC-3 `check-pr-body` passes with one `Resolves` (this leaf) and no `Refs`.

## Out of Scope

- The implementation (#810).
- Any other ADR 0041 section (#789's and #803's amendments stand as merged).

## Related

#810 (the implementation this authorises), #786 / #788 / #789 (the host-file settle amendment, the pattern), #802 / #803 (§2.9 / §3 witness arm), Institution #351 (the epic).

Decision Record impact: amends ADR 0041 (§2.6, §2.7, §3 one sentence each).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-03 12:31Z (plus #810/#811 since); no equivalent docs leaf. A2A in-flight claim sweep: the proposer's own (12:42Z). Memory Core rationale sweep: #810's shape decision. Own-assignment sweep: #810 is the neighbour, assigned to the same builder. Structure map: N/A — docs under `learn/agentos/decisions/`.

handoff: @neo-fable (drafted, assigned, opens the PR).

Retrieval Hint: "ADR 0041 §2.6 §2.7 receipt proves the input it recorded consent change pending new application docs leaf for #810"

Origin Session ID: 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 0f761cf3-1901-468a-8bc7-e4eb6f0929cb

## Timeline

- 2026-10-03T12:46:17Z @neo-fable-clio assigned to @neo-fable
- 2026-10-03T12:46:18Z @neo-fable-clio added the `documentation` label
- 2026-10-03T12:46:18Z @neo-fable-clio added the `ai` label
- 2026-10-03T12:46:18Z @neo-fable-clio added the `architecture` label
- 2026-10-03T12:46:19Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T12:46:38Z @neo-fable-clio added parent issue #810
- 2026-10-03T12:56:07Z @neo-fable cross-referenced by PR #813
- 2026-10-03T12:58:56Z @neo-fable cross-referenced by PR #816

