---
id: 914
title: 'Fleet verbs for the operator''s own inbox: read, mark read, reply, resolve'
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-07T11:37:52Z'
updatedAt: '2026-10-07T12:52:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/914'
author: neo-opus-vega
commentsCount: 0
parentIssue: 551
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-07T12:52:53Z'
---
# Fleet verbs for the operator's own inbox: read, mark read, reply, resolve

## Context

This is the Brain leaf of neomjs/neo-agent-institution#551, row 4 of FM v1. Next action per the steward disposition ([6036744624](https://github.com/neomjs/neo-agent-institution/issues/551#issuecomment-6036744624)): Brain Fleet verbs first, then the Institution UI. On 2026-10-07 the operator reported that the installed Mailbox shows previews only: he cannot open a message's full body, mark his own messages read, or reply from a message.

The Memory Core half shipped with #860 (closes #859; merged 2026-10-05 as `f5d3253d`, running on the plane). The Fleet wire does not reach it.

## The Problem

At `2d839fc1`, the Fleet wire carries one mailbox write and no mailbox read for the viewer's own messages:
- No verb reads a message body for the viewer.
- No verb marks the viewer's own message read.
- `composeOperatorMessage` whitelists `to`, `subject`, `body`, `priority`, `wakeSuppressed` and `relatedTickets`, but not `inReplyTo`. A reply therefore cannot link to the message it answers.
- No verb moves a Task the viewer holds.

## The Architectural Reality

- `FleetControlBridge` verbs delegate to injected writers and sources. `composeOperatorMessage` (L1107) routes through `composeWriter.addMessage`; the sender is never a parameter, because `MailboxService` resolves it from the transport-stamped request identity. `wireOperatorComposeWriter.mjs` installs the writer at the entry.
- The wire vocabulary is `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs`). Every verb has a row in both `fleetServerPolicy.mjs` tables: phase (L41+) and class (L93+).
- The Memory Core methods, each under the request identity:
  - `getMessage({messageId})` (L3917) enforces `CAN_READ_INBOX_OF`, which admits the viewer's own inbox, and writes no receipt.
  - `markRead({messageId, all, includeUnseen})` (L4189) writes only the caller's receipts.
  - `transitionTask({taskId, newState, expectedCurrentState})` (L4652) takes the id of the message node that carries the Task; #860's contract decides which moves the recipient may make.
- `fleetMailboxMirror` (L1158) deliberately carries no mutation for an **agent's** mailbox. These verbs act only on the viewer's **own** messages.

## The Fix

1. `composeOperatorMessage` admits `inReplyTo`. A non-string value is refused before the writer, as `relatedTickets` is today.
2. `fleetOwnMessage({messageId})`, class `read-observe`: the viewer's `getMessage`.
3. `markOwnMessageRead({messageId})`, class `lifecycle-write`: the viewer's `markRead({messageId})`. A single message only; the verb has no bulk path.
4. `transitionOwnTask({messageId, newState, expectedCurrentState})`, class `lifecycle-write`: the viewer's `transitionTask`. A refusal comes back as Memory Core answered it.
5. Wire the three new verbs beside the compose writer: wire methods, both policy rows, the plane client and the in-process entry. A primitive that is not wired leaves only its own verb `not-wired`.

The names say *own*, not *operator*: they act on the viewer's own inbox, whoever the viewer is.

## Acceptance Criteria

- [ ] AC-1: compose carries `inReplyTo` to `addMessage`, and a non-string `inReplyTo` is refused before the writer runs (unit).
- [ ] AC-2: `fleetOwnMessage` routes exactly `{messageId}` to the viewer's `getMessage`, which writes no receipt (`MailboxService` L3917–3990) (unit).
- [ ] AC-3: `markOwnMessageRead` routes exactly `{messageId}`; `all` and `includeUnseen` never cross (unit).
- [ ] AC-4: `transitionOwnTask` passes the message id as `taskId` with the requested state, and returns a refusal exactly as Memory Core answered it (unit).
- [ ] AC-5: each new verb answers `not-wired` when its primitive is absent, and the wire vocabulary and both policy tables list it (unit).

## Out of Scope

- The Institution Mailbox UI: neomjs/neo-agent-institution#551's second PR.
- Reading peer-to-peer messages, which is D#19440.
- Bulk mark-read.
- Auto-resolving requests from evidence (v1.x, recorded on #551).

## Related

neomjs/neo-agent-institution#551 (parent outcome) · #860 · #859

Live latest-open sweep, A2A sweep, MC sweep and own-assignment sweep: see the creation notes below.

Retrieval Hint: "Fleet verbs operator own inbox get_message mark_read transition_task inReplyTo"

Origin Session ID: 7dcf11bd-1a91-43b6-affd-6b4dde2c4088


## Timeline

- 2026-10-07T11:37:53Z @neo-opus-vega added the `enhancement` label
- 2026-10-07T11:37:53Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-07T11:37:53Z @neo-opus-vega added the `ai` label
- 2026-10-07T11:37:55Z @neo-opus-vega added parent issue #551
- 2026-10-07T11:45:45Z @neo-opus-vega cross-referenced by PR #915
- 2026-10-07T11:46:19Z @neo-opus-grace cross-referenced by #414
- 2026-10-07T11:51:04Z @neo-opus-vega referenced in commit `85a57d1` - "docs(fleet): the own-Task verb's JSDoc describes behaviour, not a ticket (#914)"
- 2026-10-07T12:09:08Z @neo-opus-vega cross-referenced by #551
- 2026-10-07T12:38:27Z @neo-opus-vega referenced in commit `6d34c0a` - "fix(fleet): a refused Task move reaches the operator as its code and reason, never a bare failure (#914)

MailboxService.transitionTask throws for a move its caller may not make, and the dispatcher turned that throw into a bare operation-failed. The bridge now recognizes the primitive's refusal shapes, in process and in a plane's tool-error text, and answers them as {success: false, code, reason} with a fixed reason. Any other throw stays generic. A spec pins each shape to the primitive's source."
- 2026-10-07T12:52:53Z @tobiu referenced in commit `f750655` - "feat(fleet): the viewer's own inbox gets a body read, a receipt, a Task move and reply linkage on the Fleet wire (#914) (#915)

* feat(fleet): the viewer's own inbox gets a body read, a receipt, a Task move and reply linkage on the Fleet wire (#914)

* docs(fleet): the own-Task verb's JSDoc describes behaviour, not a ticket (#914)

* fix(fleet): a refused Task move reaches the operator as its code and reason, never a bare failure (#914)

MailboxService.transitionTask throws for a move its caller may not make, and the dispatcher turned that throw into a bare operation-failed. The bridge now recognizes the primitive's refusal shapes, in process and in a plane's tool-error text, and answers them as {success: false, code, reason} with a fixed reason. Any other throw stays generic. A spec pins each shape to the primitive's source."
- 2026-10-07T12:52:53Z @tobiu closed this issue

