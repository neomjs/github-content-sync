---
id: 240
title: 'The claude-courier spool has no drain: a route that selects it accepts wakes and discards them'
state: CLOSED
labels: []
assignees:
  - neo-opus-grace
createdAt: '2026-08-29T21:56:40Z'
updatedAt: '2026-08-30T06:50:13Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/240'
author: tobiu
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
closedAt: '2026-08-30T06:50:13Z'
---
# The claude-courier spool has no drain: a route that selects it accepts wakes and discards them

The `claude-courier` transport is wired end to end as a selectable adapter — dispatched in
`localWakeAdapters.mjs`, listed in `receiver.mjs`, accepted by the MCP schema and
`WakeSubscriptionService` validation — but **nothing drains its spool**.

Measured at `origin/dev @ d040805`, whole repo, excluding the module's own file:

| export | external callers |
|---|---|
| `deliverClaudeCourier` (producer) | `localWakeAdapters.mjs` **+ spec** |
| `listOutboxEntries` | spec only |
| `completeOutboxEntry` | spec only |
| `writeCourierReceipt` | spec only |
| `RECEIPT_OUTCOMES` | none |

The producer row is the positive control: the same search finds a production caller when one exists,
so the empty rows are absence rather than a broken instrument.

## Why this blocks its parent rather than being a follow-up

Select `claude-courier` on any route today and the receiver accepts the wake, writes a spool entry,
reports `courier-spool-accepted`, and nothing ever reads it. That is
**accepted-then-silently-discarded** — the exact failure class neomjs/neo-agent-brain#30 exists to
eliminate, and worse than the `osascript` path it replaces, which at least emits `-2700` when it loses
a focus race. It is also reachable by the single most obvious action anyone picking up #30 would take:
flip a route to the new adapter.

## The addressing hazard the drain has to close

`deliverClaudeCourier` records `targetPid` and `targetSocket` as they looked at **enqueue** time. Both
are ephemeral — a seat can exit and restart between spooling and draining, and pids are reused. A
drain that addressed a wake by a snapshotted pid would deliver one seat's coordination traffic into
whatever now holds that number: **misdelivery, which is worse than non-delivery.**

The stable half of the binding is the route-owned cwd. Snapshot that; re-resolve the session from it
at send time. A session `name` must never be persisted at all — names are derived per session and a
cached one can come to name a different seat.

## Shape

Delivery goes through `SendMessage`, a Claude Code **tool** that no host process can call, so the
courier is a long-lived Claude session and the deliverable is the surface it drives: a plan step and
an outcome-recording step, with every filesystem invariant in testable Node code.

## Acceptance Criteria

- [ ] A drain pass re-resolves the target session at send time from the route-owned cwd binding; the
      spooled pid and socket are never used as an address.
- [ ] No session `name` is persisted across the spool hop — it is read from the live registry at send
      time and discarded.
- [ ] An entry that cannot be addressed — no live session, an ambiguous tie, or an entry carrying no
      cwd binding — is reported with a typed reason and never sent to a guessed target.
- [ ] A delivery outcome is written as a receipt **before** the spool entry is retired, so a crash
      between the two re-drains rather than losing the proof.
- [ ] A transient failure keeps its spool entry instead of discarding the wake.
- [ ] An entry the producer spools is drainable end to end, proven by an arm that runs the producer
      and then plans a pass over what it wrote.

## Contract Ledger

Added 2026-08-30 at reviewer request (PR #241 RA-2). This is the surface the implementation is held
to; the PR reconciles against these rows.

### Spool entry — producer-owned fields the drain depends on

| field | type | authority | contract |
|---|---|---|---|
| `eventId` | String | receiver | Correlates entry ↔ receipt. **Also authenticates completion**: a completion naming a different id is refused. |
| `targetIdentity` | String | route | The seat the wake is for. Absent ⇒ `unroutable-entry`. |
| `targetCwd` | String | route (`courierIdentityCwdMap`) | The **stable** half of the binding. The drain re-resolves the live session from this at send time. Absent ⇒ `unroutable-entry`; never inferred from `targetIdentity`. |
| `targetPid` / `targetSocket` | Number / String | enqueue-time snapshot | **Telemetry only, never an address.** Both go stale across a restart and a pid can be reused; addressing by either turns non-delivery into misdelivery. |
| `digest` | String | receiver | The message body, verbatim. Non-string ⇒ `unroutable-entry`. |
| `subject`, `enqueuedAt`, `subscriptionId` | String | receiver | Informational; surfaced in the plan for triage. |

### Plan row — what `list` returns per spooled entry

| field | contract |
|---|---|
| `status` | Exactly one of `PLAN_STATUSES`: `ready`, `no-live-session`, `ambiguous`, `unaddressable-session`, `unreadable-entry`, `unroutable-entry`. Only `ready` may be sent; every other value is a **blocked** row counted in `blockedCount`. |
| `handle` | **Opaque entry name, never a path.** The only accepted completion target. |
| `eventId`, `identity`, `subject`, `ageMs` | Correlation and triage. Present on every row **except** `unreadable-entry` — see the row-shape table below. |
| `sessionName` | Present only on `ready`. Read from the live registry **at plan time and never persisted** — names are derived per session and a cached one can name a different seat. |
| `pidAtEnqueue`, `pidNow`, `pidChanged` | Staleness telemetry. `pidChanged` is not an error: the seat restarted and the wake is still for that seat. |
| `message` | The digest to send. |
| `detail` | Typed reason on every non-`ready` status. |

**Row shape by status.** Not every row carries every field, because a row can be produced before the
entry is legible:

| status | fields present | why |
|---|---|---|
| `ready` | full row + `sessionName`, `pidNow`, `pidChanged`, `message` | Addressable; the only status the courier may send. |
| `no-live-session`, `ambiguous`, `unaddressable-session`, `unroutable-entry` | full row + `detail` | The entry parsed, so its correlation fields are known; only addressing failed. |
| `unreadable-entry` | `handle`, `eventId: null`, `status`, `detail` **only** | The file could not be parsed, so no correlation field exists to report. The row exists precisely so the file is visible rather than silently absent — it is reported, never deleted. |

### CLI

| command | inputs | contract |
|---|---|---|
| `list` | `[--outbox-dir]` | Read-only. Writes no receipt, retires nothing, claims nothing. An entry stays in the outbox until an explicit outcome is recorded, so a courier that dies mid-pass re-drains rather than loses. |
| `complete` | `--handle`, `--event-id`, `--outcome`, `[--detail]`, `[--outbox-dir]`, `[--receipts-dir]` | Destructive. **Both** facts are proven before any write: the handle resolves to a regular direct `.json` child of the configured outbox, and that entry's persisted `eventId` equals `--event-id`. A refusal is exit `2` with a reason and leaves no trace. |

### Receipt

| aspect | contract |
|---|---|
| `outcome` | Exactly one of `RECEIPT_OUTCOMES`: `delivered`, `held`, `expired`, `refused`, `error`. Anything else throws — it is a call-site error, not data to persist. |
| shape | `{schemaVersion, eventId, outcome, detail, at}`, one file per sanitized event id, replaceable latest-outcome. |
| ordering | **Receipt is written before the entry is retired.** A crash between the two re-drains and overwrites a latest-outcome receipt; the reverse order would lose the proof and leave the wake both undelivered and unrecorded. |
| retention | `error` **retains** its spool entry — the seat may be back next pass, and discarding it here would be this transport committing the failure it exists to remove. Every other outcome retires. |

### Boundaries this ledger asserts

- The drain never decides an entry is undeliverable and drops it; only an explicit recorded outcome retires anything.
- No absolute path is ever accepted as authority, and none is handed back by `list`.
- No session `name` crosses the spool hop.

## Out of Scope

- Migrating any live route to `claude-courier`. Parent neomjs/neo-agent-brain#30 owns that, and it
  stays gated on a running courier plus a positive receipt.
- The Codex half of #30 (`@neo-gpt`, `@neo-gpt-emmy` → `codex-app-server`). Same parent, different
  transport: `SendMessage` cannot reach a Codex seat, so the drain does nothing for them.
- Ingress-based delivery — neomjs/neo-agent-brain#50 owns clients without host-reachable listeners.

Related: neomjs/neo-agent-brain#30



## Timeline

- 2026-08-29T21:56:42Z @tobiu assigned to @neo-opus-grace
- 2026-08-29T22:00:16Z @tobiu cross-referenced by PR #241
- 2026-08-29T22:00:44Z @tobiu cross-referenced by #30
- 2026-08-30T01:10:19Z @neo-opus-grace referenced in commit `1f17767` - "fix(wake): bind courier completion to a proven entry and stop hiding blocked work (#240)

Review round 1 (@neo-gpt-emmy), four required actions. All four were real.

RA-1. `complete --file` accepted an arbitrary path independently of --event-id:
it wrote that event's receipt, then deleted whatever the path named. The reviewer
ran the falsifier -- non-outbox file plus a forged event id, exit 0, receipt
written, victim deleted. Completion now takes an OPAQUE handle, never a path, and
proves two independent facts before any write: the handle resolves to a regular
direct .json child of the configured outbox, and that entry's persisted eventId
equals --event-id. A refusal leaves no receipt and retires nothing. `list` no
longer hands back an absolute path at all, so there is no path to treat as
authority.

RA-2. Contract Ledger added to #240 covering targetCwd, plan statuses and row
shapes, both CLI surfaces, receipt outcomes, and the receipt-before-retire and
transient-retain semantics.

RA-3. planOne promoted a resolved session with name '' to 'ready', but
SendMessage addresses BY NAME, so the courier got a plan it could not execute.
Now a typed 'unaddressable-session' row naming its own cause.

RA-4. listOutboxEntries caught parse failures and FILTERED the row out, so a
corrupt spool file stayed queued forever while the pass reported nothing to
disposition -- undeliverable and invisible at once, the exact silent loss this
transport exists to remove. It is now reported as a typed 'unreadable-entry' plan
row, and planning still never deletes anything.

Seven new arms; four mutation controls, one per behavioural RA, each red on the
reviewed defect and green on the fix."
- 2026-08-30T06:50:13Z @tobiu closed this issue

