---
id: 977
title: 'ADR 0042: the FM publish gate — request contract, custody, the switch'
state: OPEN
labels:
  - documentation
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-10T22:29:29Z'
updatedAt: '2026-10-10T22:29:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/977'
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
# ADR 0042: the FM publish gate — request contract, custody, the switch

## Context

Graduated from [D#19500](https://github.com/neomjs/neo/discussions/19500) (outbound as a measured experiment) at its §6.2 quorum, 2026-10-10 22:21Z, with `Decision Record: REQUIRED`. The operator's design (10-10): a peer never publishes — it sends the operator one typed A2A request; the operator's Fleet Manager holds one switch per peer × channel; a Fleet-side executor publishes under the account's own credential. That adds a contract and a custody surface the existing records do not cover, and Ada's `STEP_BACK` re-read ([18855553](https://github.com/neomjs/neo/discussions/19500#discussioncomment-18855553)) named the three decisions the record must make. This ADR precedes the first publish from the gate; the leaves that implement it (the request category, the executor, the Institution's card) cite it.

## The Problem

Three decisions have no home today: (1) **the request category's contract** — its fields, its version, and who may send it to whom; (2) **the custody rule** — platform tokens live only in Fleet custody and never appear in a payload, a log or a Task; (3) **the switch** — its states, who may change it, and the promotion rule. Without the record, the implementers of three leaves in two repositories would read the Discussion's eight folds as their authority — exactly the "two authorities" Sophie's deferral ([18856006](https://github.com/neomjs/neo/discussions/19500#discussioncomment-18856006)) refused.

## The Architectural Reality

- `learn/agentos/decisions/` holds 0001–0041; this is **0042**. ADR 0036 (durable community-activity authority) governs the *reading* half (replies and mentions as community activity — occurrence, attention, explicit Task claim; popularity excluded) and is not amended. Fleet custody under `<userData>/brain/fleet` is the shape of neomjs/neo-agent-institution#345 / #346 (registrations, credentials and keys outside the replaceable bundle) — if a decision record exists for it, 0042 amends it rather than restating it; the sweep found none under `learn/agentos/decisions` naming custody.
- The mailbox already mints the identity (`ai/services/memory-core/MailboxService.mjs:2809`, `MESSAGE:<uuid>`), owns Task transitions and their RBAC (`:2939`), and the Fleet bridge routes `transitionTask` by that id (`ai/services/fleet/FleetControlBridge.mjs:1393–1408`); the embedded `task.id` is optional caller metadata. Effects with reconcile-required outcomes exist (`ai/services/fleet/hostEffects.mjs`, `verifyEffect.mjs`).
- Platform facts the record binds (read 2026-10-10, sources in the Discussion's §8): X posts cost credits per request (a post with a URL $0.20), `POST /2/tweets` needs OAuth 2.0 user context with `offline.access` for a refresh token, 100 posts / 15 min per user; LinkedIn page posting needs the Community Management API behind an access application, 60-day tokens, refresh tokens for approved partners only; LinkedIn's agreement bars a member profile per peer; X's automation rules state no exemption for human-approved API posting — the reply classification is unresolved.

## The Fix

`learn/agentos/decisions/0042-fm-publish-gate.md`, in the house ADR shape (Status · Context · Decision · Consequences · Falsifiers), deciding:

1. **The request contract** — one category (`requestPublish`; `channel` ∈ {`x`, `linkedin`}), versioned; fields: channel, account, text, media references, hypothesis, format (post · link-reply · reply), optional `inReplyTo`, the reviewer's attestation bound to a revision digest; **identity** = the mailbox's returned `MESSAGE:` id, never an embedded `task.id`; **revisions** immutable with a content digest over account, channel, text, media and reply target — an approval or attestation binds to one digest, an Edit is a new revision inheriting neither; **states**: the Task envelope's `Submitted` → `Completed` (with the provider id) | `Rejected`, over a durable publication state `queued · publishing · published · refused · unresolved`; **who may send it to whom**: a Fleet seat to the operator identity only.
2. **The custody rule** — platform tokens live only in Fleet custody (`<userData>/brain/fleet`), placed by the operator's one-time authorization; never in a payload, a log, a Task or a seat; no seat calls a platform API.
3. **The switch** — Fleet-side state behind an operator-authenticated write (component state reachable through the Neural Link is a view, never the switch); states `approve each · approve the batch · auto-approve after a cross-family review attestation · auto-approve`; read at publish time; every change audited (who · when · from · to); promotion only by the operator's hand after the FM's recorded proposal — never a counter.
4. **Consequences and falsifiers** — at-most-once publish (`publishing` recorded before the platform call; reconciliation against the account's recent posts before any retry; ambiguous → `unresolved`); the generic `Completed` transition carries no publish-success proof — the receipt is the provider id and a read-back; the executor's reply verb stays disabled until X's classification of an operator-approved reply is established; the ADR is falsified if a seat ever needs a social write capability to make the gate work.

**Decision Record impact:** `aligned-with` ADR 0036 (reading half untouched); `amends` the Fleet custody record if one exists, else `none`. Status `Accepted` on merge; the implementing leaves cite §.

## Acceptance Criteria

- [ ] AC-1 — `learn/agentos/decisions/0042-fm-publish-gate.md` exists in the house shape and decides the three things above with the field list, the identity rule, the revision rule, the custody rule, the switch's states and promotion rule, and the at-most-once consequence — each as a sentence an implementer can cite.
- [ ] AC-2 — It names ADR 0036 as the reading half's authority and either amends the Fleet custody record by number or states that none exists.
- [ ] AC-3 — It records the unresolved reply classification as a falsifier with the primary source to verify against, not as a settled fact.
- [ ] AC-4 — The cross-family reviewer (a GPT seat) answers from the record alone: who may send what to whom, where a token may live, who may change the switch and when — recorded in the review.
- [ ] AC-5 *(post-merge)* — D#19500's `[GRADUATED_TO_TICKET]` line and the three implementing leaves link the merged record.

## Out of Scope

The implementation (the request category leaf, the executor leaf, the Institution card leaf); the reading provider under ADR 0036; the experiment's run (neomjs/neo#19575); any platform entitlement.

## Avoided Traps

- **Restating the custody record** — 0042 amends by number or records absence.
- **Deciding the reply classification by inference** — a falsifier with a source, not a decision.
- **A switch in component state** — one Neural Link write away from any seat (Ada).
- **The embedded `task.id` as the key** — caller metadata, not identity (Euclid).

## Related

D#19500 (the design, the signal ledger) · neomjs/neo#19575 (the experiment) · the request-category leaf and the executor leaf (filed beside this one) · the Institution card leaf · #123 (Leaf 3 Social-MCP, reshaped by this record) · neomjs/neo-agent-institution#345, #346 (custody) · ADR 0036 · #345/#346's shape in `<userData>/brain/fleet`.

## Sweeps (ticket-create §1)

Live latest-open sweep: the latest 20 open issues of `neomjs/neo-agent-brain` at 2026-10-10 22:27:34Z (newest #969); no ADR or publish-gate ticket. A2A in-flight claim sweep: the mailbox read continuously through this sitting; no claim on this scope. Memory Core rationale sweep: the Sandbox's own folds; the April Social-MCP idea lives in #123 Leaf 3, which this reshapes. Own-assignment sweep (Brain): #850, #50, #51, #53 — none. Structure map (§1c, `npm run ai:structure-map -- --files --loc`): a docs file under `learn/agentos/decisions/` (0001–0041 present) — N/A for `ai/`. §1d: D#19500 at quorum (22:21Z); this ticket is its Decision Record target. Build order: after the v13.2 cut, before the first publish from the gate.

Origin Session ID: 7885601f-b39c-4b4b-b246-f768b2157a7c
Retrieval Hint: "ADR 0042 FM publish gate requestPublish contract custody switch at-most-once"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 7885601f-b39c-4b4b-b246-f768b2157a7c

## Timeline

- 2026-10-10T22:29:29Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-10T22:29:31Z @neo-fable-clio added the `documentation` label
- 2026-10-10T22:29:31Z @neo-fable-clio added the `ai` label
- 2026-10-10T22:29:31Z @neo-fable-clio added the `architecture` label
- 2026-10-10T22:29:31Z @neo-fable-clio added the `agent-os` label

