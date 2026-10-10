---
id: 978
title: 'MailboxService: the requestPublish category and its revisioned payload'
state: OPEN
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-10T22:30:04Z'
updatedAt: '2026-10-10T22:30:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/978'
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
---
# MailboxService: the requestPublish category and its revisioned payload

## Context

Graduated from [D#19500](https://github.com/neomjs/neo/discussions/19500) (§6.2 quorum 2026-10-10 22:21Z): the operator's design makes the Fleet Manager the publish gate — a peer sends the operator one typed A2A request, the FM decides. This leaf is the **request half** in the Brain: the message category, its payload contract and its states, so that the Institution's request card and the Fleet-side executor consume one authority. It implements ADR 0042 (filed beside this leaf) and reshapes Brain #123's Leaf 3 (Social-MCP: no MCP server for seats — a request category instead).

**unowned-rationale:** peers self-select after the v13.2 cut and the FM v1 walks; the ADR leaf merges first.

## The Problem

A2A messages already carry a Task envelope, tagged concepts and a sender identity, and the mailbox already mints the id and owns transitions — but nothing types a *publish request*: which fields it carries, which seat may send it to whom, how a payload is kept immutable so an approval can bind to it, and what durable state records what happened to it after the operator acted. Without the contract, the card and the executor each invent a shape.

## The Architectural Reality

- `ai/services/memory-core/MailboxService.mjs:2809` mints `MESSAGE:<uuid>` at send and returns it — **the request's identity** (Ada's re-read; Euclid's source read: the embedded `task.id` is optional caller metadata and must not be the key). `:2939`: Task state transitions and their RBAC are owned by `transitionTask`; the Fleet bridge routes them by message id (`ai/services/fleet/FleetControlBridge.mjs:1393–1408`). Wake-suppression rules already key on `taggedConcepts` (`:389–431`) — the category is another typed concept, not another transport.
- The operator's inbox in the FM (neomjs/neo-agent-institution#551) reads Tasks addressed to the operator under his viewer identity; the request card (the Institution leaf) extends it.
- The generic `Completed` transition proves no publish (Euclid): the publication receipt is the executor's — provider id and read-back — recorded in a publication state beside the Task state.

## The Fix

In `MailboxService.mjs` and its OpenAPI/contract surface (one module, one validation seam):

1. **The category** `requestPublish` as a typed Task to the operator identity: `channel` ∈ {`x`, `linkedin`}, `account`, `text`, `media` (references), `hypothesis`, `format` ∈ {`post`, `link-reply`, `reply`}, optional `inReplyTo` (a provider id), `attestation` (the reviewer's review-message id, bound to a revision digest). A seat may send it only to the operator identity; the mailbox refuses other recipients and malformed payloads with a reason.
2. **Revisions** — the payload is immutable once sent; a content digest over account, channel, text, media and reply target is computed by the mailbox and returned with the id; an Edit from the operator's card is a **new revision** under the same request id, with its own digest, inheriting neither approval nor attestation.
3. **States** — the Task envelope keeps `Submitted` → `Completed` (with the provider id) | `Rejected` (with the operator's reason); beside it a durable **publication state** the executor owns and the mailbox stores: `queued · publishing · published · refused · unresolved`, with the text as requested and the text as published under the id (the edit metric).
4. **No wake by default** for the category (the operator reads his inbox; the FM renders); the Task envelope's `expiresAt` applies.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `add_message` with `taggedConcepts: ['requestPublish']` + typed payload | ADR 0042 §1; `MailboxService.mjs:2809` (id) | validates fields, recipient = operator identity, computes the revision digest, returns `{messageId, revision}` | refuses with a named reason | OpenAPI + the ADR | unit: accept/refuse arms per field, per recipient |
| revision on Edit | ADR 0042 §1 | new revision under the request id, new digest, no inherited approval/attestation | — | OpenAPI | unit: an Edit's digest differs; approval keyed to the old digest does not match |
| publication state | ADR 0042 §4 | `queued → publishing → published | refused | unresolved`, written by the executor through one verb | `unresolved` on ambiguity | OpenAPI | unit: the state machine's legal moves; `Completed` without `published` is refused |

**Decision Record impact:** `depends-on` ADR 0042.

## Acceptance Criteria

- [ ] AC-1 — A `requestPublish` Task with every field validates and returns the mailbox id and the revision digest; a missing or unknown field, an unknown channel or format, or a recipient other than the operator identity is refused with a reason (unit arms per case).
- [ ] AC-2 — A revision's digest changes when any of account, channel, text, media or `inReplyTo` changes; an approval or attestation recorded against the old digest does not match the new revision (unit).
- [ ] AC-3 — The publication state admits only the legal moves and stores both texts; `Completed` is refused while the publication state is not `published` (unit).
- [ ] AC-4 — The OpenAPI describes the category, the payload, the revision and the publication state in the same words as ADR 0042; the Fleet bridge's `transitionTask` routing needs no change for the category (asserted by a bridge spec).
- [ ] AC-5 — No token, credential or platform secret appears in the payload schema (a negative schema test).

## Out of Scope

The executor (its own leaf); the card and the switch (Institution); any platform call; the reading provider.

## Avoided Traps

- **A per-seat counter as the key** — two harness instances of one handle collide; the mailbox's id is the identity (Ada).
- **`task.id` as the key** — optional caller metadata (Euclid).
- **An approval that survives an Edit** — binds to the digest, not the request (Sophie).
- **`Completed` as a publish receipt** — the receipt is the executor's provider id (Euclid).

## Related

D#19500 · ADR 0042 (filed beside this) · the executor leaf (filed beside this) · the Institution card leaf · neomjs/neo#19575 (the experiment) · #123 (Leaf 3, reshaped) · neomjs/neo-agent-institution#551.

## Sweeps (ticket-create §1)

Live latest-open sweep: the latest 20 open issues of `neomjs/neo-agent-brain` at 2026-10-10 22:27:34Z (newest #969); none on publishing. A2A in-flight claim sweep: the mailbox read continuously through this sitting; no claim. Memory Core rationale sweep: the Sandbox's folds. Own-assignment sweep (Brain): #850, #50, #51, #53 — none. Structure map (§1c, `npm run ai:structure-map -- --files --loc`): `ai/services/memory-core` (35 files) owns `MailboxService.mjs` — the category lives there, no new file; sibling precedent: the wake-suppression tag rules at `:389–431`. §1d: D#19500 at quorum; this ticket is a `[GRADUATED_TO_TICKET]` target.

Origin Session ID: 7885601f-b39c-4b4b-b246-f768b2157a7c
Retrieval Hint: "requestPublish mailbox category revision digest publication state operator inbox"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 7885601f-b39c-4b4b-b246-f768b2157a7c

## Timeline

- 2026-10-10T22:30:06Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T22:30:06Z @neo-fable-clio added the `ai` label
- 2026-10-10T22:30:06Z @neo-fable-clio added the `architecture` label
- 2026-10-10T22:30:06Z @neo-fable-clio added the `agent-os` label

