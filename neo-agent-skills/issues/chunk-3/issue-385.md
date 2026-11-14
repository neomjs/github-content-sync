---
id: 385
title: A successor session leaves no marker for the rotation to read
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T17:03:48Z'
updatedAt: '2026-11-14T18:23:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/385'
author: neo-opus-vega
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
closedAt: '2026-11-14T18:23:53Z'
---
# A successor session leaves no marker for the rotation to read

## Context

The rotation lane of neomjs/neo-agent-brain#1383 (PR neomjs/neo-agent-brain#1693) keys two decisions on the Memory Core session id. `newer-session` refuses a rotation when a memory from an id other than the receipt's appears, and `complete` marks one done on the same signal after the request. Ada's specimen from the live plane (A2A `dffed068`, Nov 14):

- Clio's receipt `6702a2ed` went out at 15:55:53Z, and the lane refused it with `newer-session` at 15:57:15Z.
- Clio's own sunset turn had written its diary 19 s after the receipt under a new Memory Core id (`3c9a08d0`, the receipt's was `c3373e53`).
- Later wakes into the same ended transcript wrote under a third (`9ed19770`).

A Memory Core id rotates inside one transcript (a reconnect, or a plane cut: the Avoided Trap of #377), so it cannot tell a successor from the old transcript.

## The Problem

Nothing tells the lane that a successor exists. #377 gave the old transcript its marker, `[sunset rescinded]`; the successor side has none, so the lane fell back to the id.

## The Architectural Reality

- `context-recovery` step 1a (`references/context-recovery-workflow.md`) reads the latest handover in full. The transcript that wrote it rescinds if it resumes work, and "a successor session never writes one" (#377).
- The rotation keys on the newest self-DM, so any self-DM after a handover cancels it (#377, Fix item 3). A successor's self-DM therefore already ends further rotation requests for its handover.
- Design authority: #377's Avoided Trap, "Keying the rescind on session id alone: a plane cut rotates the id inside one transcript", and neomjs/neo-agent-brain#1383's receipt-driven rotation, one event per receipt.

## The Fix

1. Step 1a gains one line. A session that did not write the latest handover (a rotated successor or a hand-opened one) sends a quiet self-DM `[sunset successor]` tagged `sunset-successor`, with `inReplyTo` set to the handover, before it resumes any lane. The line names its retirement: it goes when Memory Core sessions bind to the harness transcript (D#19401).
2. The package version is bumped.

The lane side is Ada's, after neomjs/neo-agent-brain#1711: the id-keyed `newer-session` goes, `complete` keys on the marker, and `buildRotationPrompt` names it (A2A `fb0390aa`).

## Acceptance Criteria

- [ ] AC-1 Step 1a carries the successor line, with its retirement trigger, and keeps "a successor session never writes a rescind".
- [ ] AC-2 `context-recovery-workflow.md` grows by that one line at most; the PR body names the net bytes.
- [ ] AC-3 The package version is bumped.

## Out of Scope

- The lane's readers and `buildRotationPrompt` (neomjs/neo-agent-brain, Ada, after #1711).
- Binding Memory Core sessions to the harness transcript (D#19401).

## Avoided Traps

- **A field stamp on the receipt:** it needs a new write path on an MCP tool, while a self-DM goes through `add_message` as it is, and the newest-self-DM rule already reads it.
- **Keying on the session id:** the trap #377 recorded, now measured on `newer-session` too.

## Related

neomjs/neo-agent-brain#1383 · neomjs/neo-agent-brain#1710 / PR #1711 · #377 · D#19401. Pilot: my own successor self-DM in reply to my handover `a55cc803` (Nov 14, 17:0xZ).

Ownership: self-assigned and in flight, so no `good first issue`.

Live latest-open + closed sweep: latest 20 open read at 2026-11-14T17:02Z, plus closed searches here (`sunset successor rotation context-recovery`: #377, its Fix and Avoided Traps read) and in neomjs/neo-agent-brain (`successor rotation newer-session`); no equivalent, no prior exclusion. Backlog: 35 open (orange); the rule's home is step 1a, which #377 named.
A2A sweep: the last 30 messages, no claim on the successor side.
MC sweep: "rotation refused because a new Memory Core session id appeared in the same transcript after the sunset receipt", 5 results; the id rotates on reconnects (10-07, 10-09, 10-31), with no prior decision on a successor marker.
Own-assignment sweep: 3 open (#5, #6, #302), none overlapping.

Origin Session ID: 4c2e4f5b-144f-4493-8fe0-114a5597d003
Retrieval Hint: "successor session marker sunset rotation newer-session memory core id rotates inside one transcript"

## Timeline

- 2026-11-14T17:03:49Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T17:03:51Z @neo-opus-vega added the `enhancement` label
- 2026-11-14T17:03:51Z @neo-opus-vega added the `ai` label
- 2026-11-14T17:03:51Z @neo-opus-vega added the `agent-os` label
### @neo-opus-vega - 2026-11-14T17:05:22Z

**Contract Ledger (T3)**

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| A quiet self-DM `[sunset successor]` tagged `sunset-successor`, sent with `inReplyTo` the latest handover | `context-recovery` step 1a (`references/context-recovery-workflow.md`), beside #377's `sunset-rescinded` | Sent once, before any lane resumes, by a session that did not write the latest handover (a rotated or a hand-opened one). The rotation's reader skips it when it looks for the receipt, and neomjs/neo-agent-brain#1383's `complete` keys on it (neomjs/neo-agent-brain#1712, Ada). Until #1712 lands, the marker, as the newest self-DM, ends the rotation outright (#377, Fix item 3) | No marker (a seat on older skills, or a session that skipped recovery): nothing infers a successor, and the lane's operator-prompt guard, keyed on the identity's turn presence, still refuses a rotation the operator started by hand | Step 1a's line | Pilot `8a2e1490` (in reply to `a55cc803`, Nov 14); Ada's specimen `dffed068` |

Surface anchors checked: `add_message` takes `inReplyTo` and `taggedConcepts` as used (the pilot went through both); the newest-self-DM rule is #377's Fix item 3; the identity-keyed guard is `promptedTurnAt` over `AGENT_TURN_PRESENCE:<identity>:` (Ada, A2A `9eed76fc`).

*Edited Nov 14, 17:1xZ: the reader skips the marker (neomjs/neo-agent-brain#1712's AC-3, Ada's catch); before that edit this row said the marker ends the rotation as the newest self-DM, which holds only until #1712 lands.*

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-14T17:05:48Z @neo-opus-vega cross-referenced by PR #386
- 2026-11-14T17:10:24Z @neo-opus-vega cross-referenced by #1712
- 2026-11-14T17:19:04Z @neo-opus-vega referenced in commit `ad764f9` - "fix(context-recovery): the successor marker is quiet and sent once per handover (#385)

The marker carried neither condition the ledger names. It now goes out
only when the outbox step 1a reads shows no successor marker newer than
the handover, so a second recovery in the same successor sends none,
and it says wakeSuppressed, since a self-DM wakes by default."
- 2026-11-14T18:05:06Z @neo-opus-vega referenced in commit `30d3448` - "fix(context-recovery): the successor marker names its handover in the subject (#385)

No mailbox read returns inReplyTo (list_messages in every projection,
get_message), and the outbox read defaults includeArchived to false. So
step 1a reads the outbox by the sunset-successor tag with archived rows
included, and the marker carries the handover id in its subject: a second
recovery of the same successor finds its marker, and a marker for an older
handover never counts for this one."
- 2026-11-14T18:23:53Z @tobiu referenced in commit `4d2e1f9` - "feat(context-recovery): a successor marks its handover with a sunset-successor self-DM (#385) (#386)

* feat(context-recovery): a successor marks its handover with a sunset-successor self-DM (#385)

The rotation lane told a successor from the old transcript by the Memory
Core session id, which rotates inside one transcript on a reconnect, so
it refused a live rotation. Step 1a now has the successor send a quiet
self-DM tagged sunset-successor in reply to the handover, the mirror of
the rescind; the line retires when sessions bind to the transcript.

* fix(context-recovery): the successor marker is quiet and sent once per handover (#385)

The marker carried neither condition the ledger names. It now goes out
only when the outbox step 1a reads shows no successor marker newer than
the handover, so a second recovery in the same successor sends none,
and it says wakeSuppressed, since a self-DM wakes by default.

* fix(context-recovery): the successor marker names its handover in the subject (#385)

No mailbox read returns inReplyTo (list_messages in every projection,
get_message), and the outbox read defaults includeArchived to false. So
step 1a reads the outbox by the sunset-successor tag with archived rows
included, and the marker carries the handover id in its subject: a second
recovery of the same successor finds its marker, and a marker for an older
handover never counts for this one."
- 2026-11-14T18:23:54Z @tobiu closed this issue

