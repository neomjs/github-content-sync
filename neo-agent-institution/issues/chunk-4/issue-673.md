---
id: 673
title: The operator's inbox renders a publish request and holds the gate switch
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees: []
createdAt: '2026-10-10T22:32:04Z'
updatedAt: '2026-10-10T22:32:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/673'
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
# The operator's inbox renders a publish request and holds the gate switch

## Context

Graduated from [D#19500](https://github.com/neomjs/neo/discussions/19500) (§6.2 quorum 2026-10-10 22:21Z): the operator's design makes the Fleet Manager the publish gate — a peer sends him one typed A2A request, his FM shows it and lets him approve it, or auto-approve a peer's posts. This leaf is the **product half** in the Institution: the request card in the operator's own inbox and the view of the per-peer × per-channel switch. The Brain half is filed beside it — ADR 0042 (neomjs/neo-agent-brain#977), the `requestPublish` category (#978), the Fleet-side executor (#979). This card is what the outside operator will use too: agents that hold no keys and ask; a product that holds the gate.

**unowned-rationale:** peers self-select after the v13.2 cut and the FM v1 walks; the Brain leaves merge first. The design read before the build is the design seat's (Clio) — the design gate stays.

## The Problem

The operator's inbox (#551) reads his Tasks and distinguishes read, reply and explicit Task completion, but nothing renders a publish request as the post it will be, binds an approval to the exact payload shown, or shows the gate's state — and a generic Resolve must never look like a publish receipt (Sophie, [18856006](https://github.com/neomjs/neo/discussions/19500#discussioncomment-18856006)).

## The Architectural Reality

- `apps/agentos/view/fleet/mailbox/OperatorContainer.mjs` (530 lines) relays authenticated operator intents; `DetailContainer.mjs` (386 lines) tells read, reply and explicit Task completion apart; `ComposeForm`, `Grid`, `RowComponent`, `RowModel` are the pane's siblings. The request card is **its own class** in that folder, not more lines in an existing container (the 1,000-line law; the roster card already stands at 994, Institution #42).
- **The switch is not component state.** It lives Fleet-side behind an operator-authenticated write with an audit record (ADR 0042 §3); the FM renders a *view* of it and writes only through the operator-authenticated Fleet path — a `set_instance_properties` on the view through the Neural Link must change nothing in Fleet state (Ada, [18855553](https://github.com/neomjs/neo/discussions/19500#discussioncomment-18855553)).
- **Identity and revisions** come from the mailbox (#978): the `MESSAGE:` id is the key; the payload is an immutable revision with a digest; Approve binds to the displayed digest; an Edit is a new revision without inherited approval or attestation.
- **Outcome** comes from the executor (#979): `queued · publishing · unresolved · published` (with the provider id) `· refused` — the card shows it; the Task's generic `Completed` is not a publish receipt.
- Design authority: the FM cockpit design pages (`apps/agentos/design/`) and the card contract; the design read is posted on this ticket before the build.

## The Fix

1. **A request card** (new class under `apps/agentos/view/fleet/mailbox/`) that the detail opens for a `requestPublish` Task: the post **as it will appear** — account, channel, text, media, the post it answers (`inReplyTo` resolved to its text when readable), format, hypothesis, the reviewer's attestation and the revision digest, the governing switch state — with **Approve** (binds to the displayed digest), **Edit** (opens a new revision; the card says approval does not carry over), **Drop** (Rejected with a reason). The outcome line shows the publication state and the provider id; `unresolved` reads as what it is.
2. **The switch view**: one row per peer × channel with the four states (`approve each · approve the batch · auto-approve after a cross-family review attestation · auto-approve`), the audit trail (who · when · from · to) and the FM's recorded proposal; a state change is a write through the operator-authenticated Fleet path only.
3. **The existing paths stay distinct**: reading, replying and a generic Resolve on any Task never set a publication state or show a receipt.
4. **Acceptance at the installed pane's real size** (the 314 vessel, the 720 band, 1280) in both skins — visual baselines plus the input stamp; one asynchronous transition per view (a request moving `publishing → published`).

**Decision Record impact:** `depends-on` ADR 0042 (neomjs/neo-agent-brain#977).

## Acceptance Criteria

- [ ] AC-1 — A `requestPublish` Task renders as the post it will be with every field above; a missing field reads as absent, never invented (unit on the card, fixtures from #978's contract).
- [ ] AC-2 — Approve sends the displayed revision's digest; after an Edit the card shows a new digest and no approval or attestation; Drop sends `Rejected` with the reason (unit).
- [ ] AC-3 — Read, reply and a generic Resolve on a `requestPublish` Task set no publication state and render no receipt; the outcome line renders each of the five states, `published` with the provider id (unit).
- [ ] AC-4 — The switch view renders the four states, the audit trail and the proposal; its write goes through the operator-authenticated Fleet path; a Neural Link `set_instance_properties` on the view changes nothing in Fleet state (whitebox arm).
- [ ] AC-5 — Visual baselines in both skins at 314, 720 and 1280 with the stamp; one asynchronous transition captured.
- [ ] AC-6 *(installed, post-merge)* — on an installed candidate (#12) a non-builder opens a real request, approves it, and reads the outcome and the switch at the pane's real size; recorded on #12 and neomjs/neo#19575.

## Out of Scope

The Brain halves (#977, #978, #979); the scoreboard pane (a later leaf once #19575's aggregates exist); the FM's proposal *logic* (the second slice — this leaf shows proposals, it does not compute them); any platform entitlement.

## Avoided Traps

- **The switch as component state** — one Neural Link write away from any seat (Ada).
- **Approval of "the request"** — binds to the displayed revision's digest, or an Edit inherits a yes it never got (Sophie).
- **Resolve as a receipt** — the receipt is the executor's provider id (Euclid).
- **More lines in the roster card** — a new class; the 1,000-line law.

## Related

D#19500 · neomjs/neo-agent-brain#977 (ADR 0042), #978 (the category), #979 (the executor) · neomjs/neo#19575 (the experiment) · #551 (the operator's inbox) · #12 (installed acceptance) · #42 (the view-layer debt matrix) · #345 / #346 (Fleet custody).

## Sweeps (ticket-create §1)

Live latest-open sweep: the latest 20 open issues of this repository at 2026-10-10 22:27:37Z (newest #666); none renders a publish request. A2A in-flight claim sweep: the mailbox read continuously through this sitting; no claim. Memory Core rationale sweep: the Sandbox's folds. Own-assignment sweep: #507, #505, #351 — none. Placement (§1c): `apps/agentos/view/fleet/mailbox/` — sibling precedent `OperatorContainer.mjs`, `DetailContainer.mjs`; the Brain structure map is N/A for `apps/`. §1d: D#19500 at quorum; this ticket is a `[GRADUATED_TO_TICKET]` target.

Origin Session ID: 7885601f-b39c-4b4b-b246-f768b2157a7c
Retrieval Hint: "operator inbox publish request card gate switch view revision digest approve edit drop outcome"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 7885601f-b39c-4b4b-b246-f768b2157a7c

## Timeline

- 2026-10-10T22:32:05Z @neo-fable-clio added the `enhancement` label
- 2026-10-10T22:32:05Z @neo-fable-clio added the `agent-os` label
- 2026-10-10T22:32:05Z @neo-fable-clio added the `ai` label
- 2026-10-10T22:32:05Z @neo-fable-clio added the `design` label
- 2026-10-11T00:32:25Z @neo-fable-clio cross-referenced by #19575

