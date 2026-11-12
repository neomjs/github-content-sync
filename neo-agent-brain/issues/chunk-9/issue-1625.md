---
id: 1625
title: Whether the operator is away is plane state every seat can read
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T02:30:49Z'
updatedAt: '2026-11-12T04:18:06Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1625'
author: neo-opus-ada
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
blocking:
  - '[x] 1627 The fleet sets operator presence and serves it to the cockpit'
closedAt: '2026-11-12T04:18:06Z'
---
# Whether the operator is away is plane state every seat can read

## Context

Operator, 2026-11-12: a Nightshift Mode toggle in the FM, where "a state change should trigger a non-wake broadcast to tell peers if their operator is there or not". Correcting my first reading: "even when i am here, you can and should still drive. the difference is in case you have questions that peers can not help with, or need an org admin (e.g. 'Tobi, please disable CodeQL from the skills ruleset temporarily' (happened today))." The FM's toggle has its own Institution ticket.

## The Problem

- **The mode reaches seats one at a time today.** Memory Core shows the operator announcing it per seat ("nightshift until 09:59 Berlin": Euclid 10-09 and 11-07, Sophie 10-07, Mnemosyne 10-09).
- **A broadcast alone is an event, not state.** A seat that boots or recovers after the change never sees it, buried among hundreds of unread messages.
- **#95's wake policy and #842's rotation prompt have no presence fact to read.**

## The Architectural Reality

Design authority: the operator's words above. The behavior contract they set:
- **Both modes:** peers drive their lanes.
- **What the mode answers:** will an operator-only ask get a quick answer? That covers org-admin actions, credentials, merges, and questions peers cannot answer.
- **Present:** ask the operator directly.
- **Away:** such asks queue for the return (one digest, no waiting turn), and seats choose lanes that need no operator.
- **Sunset rotation** (`neomjs/neo-agent-institution#842`) runs in both modes, and its fresh session's prompt states the mode.

Today `who_is_online` carries per-identity participation and wake rows, with no operator-level record. Only the operator identity (the FM's verified viewer, @tobiu) may set this.

## The Fix

1. A plane record `operatorPresence: {mode: 'present'|'away', since, until?, setBy}`, plus a write path only the operator identity may call. The FM toggle uses it through the fleet server.
2. Readers:
   - `who_is_online` returns it at the top level;
   - the context-recovery boot read surfaces it, as a Skills follow-up once this lands;
   - #95's wake policy and #842 consume it.
3. Expiry: an `away` with `until` reads `present` once `until` passes, flagged `expired: true`, so a forgotten toggle cannot hold the fleet in `away`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| plane `operatorPresence` (new) | operator, 2026-11-12 | `present` / `away` (+ `until`), set by the operator identity only | no record reads `present`, the current assumption | the tool's OpenAPI line | unit specs: set, read, expiry, a non-operator write refused |
| `who_is_online` (existing) | this ticket | top-level `operatorPresence` | absent field = `present` | its OpenAPI description | spec |

## Decision Record impact

none

## Acceptance Criteria

- [ ] The operator identity can set `present` / `away` (+ `until`), and any other identity's write is refused.
- [ ] `who_is_online` returns `operatorPresence`, and an expired `away` reads `present` with `expired: true`.
- [ ] Specs cover set, read, expiry and refusal.

## Out of Scope

- The FM toggle (Institution ticket).
- #95's wake policy and #842's rotation; they consume this.
- The Skills text that teaches seats the contract (a follow-up once this lands).

## Related

#95, `neomjs/neo-agent-institution#842`, the Institution toggle ticket.

Live latest-open + closed sweep: Brain 154 open (red band; latest `#1624`..`#1614`) plus an org-wide open search for `operator presence` at 2026-11-12T02:28Z. Hits: #95 (consumer) and Institution #887 (unrelated); no owner of this record.
A2A in-flight claim sweep: latest 10 messages, all states, at 02:28Z; no claim.
MC sweep: `operator says whether they are present or away so peers adapt; night shift mode`, 6 results (the ad-hoc announcements above); no prior decision.
Own-assignment sweep: none on this surface.

unowned-rationale: Ready for a maintainer; a plane write path with an identity gate.

Origin Session ID: c56bdafb-bd1d-4a67-b91d-3c37931e0abb
Retrieval Hint: "operator presence away/present plane state: nightshift toggle, who_is_online operatorPresence, operator-only asks queue while away"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T02:30:51Z @neo-opus-ada added the `enhancement` label
- 2026-11-12T02:30:51Z @neo-opus-ada added the `ai` label
- 2026-11-12T02:30:52Z @neo-opus-ada added the `agent-os` label
- 2026-11-12T02:32:15Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T02:52:37Z @neo-opus-ada cross-referenced by PR #1626
- 2026-11-12T02:56:37Z @neo-opus-ada cross-referenced by #1627
- 2026-11-12T02:56:47Z @neo-opus-ada marked this issue as blocking #1627
- 2026-11-12T03:01:03Z @neo-opus-ada referenced in commit `05b73d8` - "chore(memory-core): both presence tools share one tag (#1625)

TurnPresence becomes Presence and carries set_operator_presence too, so the full-surface initialize catalog stays within its 15-line budget; the memory-core operation pin moves to 56."
- 2026-11-12T03:12:33Z @neo-opus-ada referenced in commit `ad67e88` - "feat(memory-core): operator presence becomes plane state (#1625)

The operator's present/away mode is one shared graph record. set_operator_presence (admin tier) writes it and refuses a caller whose server-stamped accountType is not human; who_is_online carries it as operatorPresence in both answers. An away past its until reads present with expired: true, and the write answers changed, so a caller broadcasts once per change."
- 2026-11-12T03:12:33Z @neo-opus-ada referenced in commit `d4ce95b` - "chore(memory-core): both presence tools share one tag (#1625)

TurnPresence becomes Presence and carries set_operator_presence too, so the full-surface initialize catalog stays within its 15-line budget; the memory-core operation pin moves to 57 (#1613's tool and this one)."
- 2026-11-12T03:43:03Z @neo-opus-ada referenced in commit `ca1b56b` - "fix(memory-core): the presence gate uses the operator predicate (#1625)

set_operator_presence now asks isHumanOperatorIdentity, the predicate the open-work intake uses since #1613: an AgentIdentity of a human account that is not external and not marked operator: false. One operator gate instead of two (Maryam's review challenge on PR #1626)."
- 2026-11-12T03:47:53Z @neo-opus-vega cross-referenced by #842
- 2026-11-12T04:18:06Z @tobiu referenced in commit `aca17ab` - "feat(memory-core): operator presence becomes plane state (#1625) (#1626)

* feat(memory-core): operator presence becomes plane state (#1625)

The operator's present/away mode is one shared graph record. set_operator_presence (admin tier) writes it and refuses a caller whose server-stamped accountType is not human; who_is_online carries it as operatorPresence in both answers. An away past its until reads present with expired: true, and the write answers changed, so a caller broadcasts once per change.

* chore(memory-core): both presence tools share one tag (#1625)

TurnPresence becomes Presence and carries set_operator_presence too, so the full-surface initialize catalog stays within its 15-line budget; the memory-core operation pin moves to 57 (#1613's tool and this one).

* fix(memory-core): the presence gate uses the operator predicate (#1625)

set_operator_presence now asks isHumanOperatorIdentity, the predicate the open-work intake uses since #1613: an AgentIdentity of a human account that is not external and not marked operator: false. One operator gate instead of two (Maryam's review challenge on PR #1626)."
- 2026-11-12T04:18:07Z @tobiu closed this issue
- 2026-11-12T04:25:27Z @neo-opus-ada cross-referenced by PR #1632
- 2026-11-12T06:29:30Z @neo-gemini-julia cross-referenced by PR #934

