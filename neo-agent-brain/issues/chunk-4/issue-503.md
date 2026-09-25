---
id: 503
title: 'A wake subscription reports itself deliverable while every dispatch fails, and the seat''s envelope writer emits a schema its own adapter refuses'
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-preview
createdAt: '2026-09-25T17:41:15Z'
updatedAt: '2026-09-25T18:03:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/503'
author: neo-preview
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
---
# A wake subscription reports itself deliverable while every dispatch fails, and the seat's envelope writer emits a schema its own adapter refuses

## Context

For **nineteen days** every wake addressed to `@neo-preview` failed at dispatch, and nothing said so. The break was found on 2026-09-25 only because a human pasted a link into the session; the two A2A messages it concerned were sitting unread in the mailbox the whole time.

Measured from the receiver's own persisted records (`~/Library/Application Support/Neo/AgentOS/wake/state/records/`), subscription `WAKE_SUB:54aaef3c-ac98-4cf1-bab3-4c007e1a6954`:

| Measure | Value |
|---|---|
| Records for this subscription | 342 |
| Last **successful** delivery | `2026-08-26T18:50:21Z` |
| Current failure mode since | `2026-09-06T12:04:47Z` |
| Records since, delivered | **114 attempted, 0 delivered** |
| `outcomeReason` on every one | `opencode-server envelope requires 'agentIdentity'` |

The upstream half is entirely healthy and stays that way: the receiver is listening, the digest is built correctly (`totalEvents: 2`, both events `sent_to_me`, correct addressee), my route is present in the receiver manifest with the correct `envelopePath`, and only one `wake-envelope.json` exists on the host. The A2A **message** path is unaffected — the messages were readable throughout. Only the wake — the interrupt that raises an idle session — is dead.

## The Problem

Two independent defects, one visible outcome.

**1. The seat's envelope writer emits a schema the adapter refuses.** `ai/daemons/wake/consumeWakeOutbox.mjs:58-66` validates the seat envelope and requires three fields: an integer `pid`, a non-empty `pidStartedAt` string, and a non-empty `agentIdentity`. The envelope this seat's writer produces carries `hostname`, `port`, `sessionId`, `projectId`, `directory`, `username`, `password`, `updatedAt` — **and none of the three**. The writer is an older generation than the reader, and the reader names its own fix: *"provision the seat identity and refresh via the SessionStart hook"*.

The contract is duplicated across two artifacts that never met: the writer's shape and the adapter's validator. Nothing makes them agree, so drift is invisible until a dispatch fails — and then only in a log directory no one reads.

**2. A subscription that cannot be reached still reports itself as armed and deliverable.** `manage_wake_subscription({action:'list'})` returns `status: "active"`, `routeDeliverable: true` for a subscription whose last 114 dispatches all failed. The status vocabulary describes **intent** — the seat wants wakes — and the health surface projects intent. It never projects the dispatch outcome, which the receiver already persists per record.

This is the same class `#17647` named from the other side: there, the wake block degrades to `daemonRunning:false, gateState:'unknown', lastPulseAt:null` when it merely *cannot read* the liveness file, so **off, dead and blind are one payload**. Here the payload is positive and wrong — **armed and deliverable, undeliverable in fact**. An instrument that cannot be wrong is not evidence, and this one is confidently wrong for nineteen days.

**Why nobody noticed, structurally:** the subscription list reads healthy, `who_is_online` reports the seat present, the orchestrator digest carries no wake-delivery observation, and healthcheck has a wake block that reports gate and heartbeat liveness — never an outcome. There is no surface on which "delivered 0 of the last 114" could appear.

## The Architectural Reality

- **The validator**: `ai/daemons/wake/consumeWakeOutbox.mjs:58-66` — `pid` (integer), `pidStartedAt` (non-empty string), `agentIdentity` (non-empty string). The throwing arm is the adapter's fail-closed contract, and it is correct as written.
- **The writer**: the seat's SessionStart hook, which materialises `~/.local/share/opencode/wake-envelope.json`. Its shape is defined by seat provisioning (`generateOpenCodeSeatConfig.mjs` owns the writer args), not by the adapter.
- **The receipt nobody reads**: the receiver persists one record per dispatch under `…/wake/state/records/`, each carrying `state`, `outcomeReason`, `dispatchStartedAt`, `dispatchFinishedAt` and the full envelope. This is the substrate's own account of what happened, and it is correct and complete.
- **The health surface**: `ai/services/memory-core/HealthService.mjs#buildWakeFeaturesBlock` projects the wake **safety gate** state and the daemon **heartbeat liveness** (with a `POLL_INTERVAL`-derived staleness threshold). It does not project delivery outcomes, and `ai/services/fleet/planeWhoIsOnlineReader.mjs` does not either.
- **The status policy already knows the shape**: `ai/services/memory-core/wakeSubscriptionStatusPolicy.mjs` opens by naming *"the failure mode this replaces (reads-active-while-undeliverable)"* — 20 decision points disagreed about what an **absent** `status` means, and the chosen rule is *absent ⇒ active*, justified as failing in the loud direction. That work fixed the **absent** case. This ticket is the **present-and-active** case, which the same reasoning does not reach: the row is unambiguous, and unambiguous is not the same as reachable.
- **Prior art, writer side**: #15684 / #15854 record the same surface failing a different way — the envelope going stale after a desktop restart because the writer did not fire, healed by hand, with a standing gap that *fresh seats are not covered* because the subscription predates the identity template. The envelope writer is a known-fragile, hand-provisioned surface; this is its schema-drift variant.

## The Fix

**Half 1 — one contract, both sides derived from it.**
- Promote the envelope's required-field contract to a single declared place that the adapter validates against *and* the writer is generated from, so a field cannot exist on one side only. The three fields' names and types are already the adapter's; what is missing is the shared declaration.
- Provision the seat identity the error asks for, so the writer emits `agentIdentity` (+ `pid`, `pidStartedAt`) on the current schema.
- A spec pins the writer's emitted shape against the adapter's validator, so the two cannot drift without a red test. This is the same shape as the `PlaneDataRootMount` guard #502 added: assert the pair, not the intent.

**Half 2 — project the delivery outcome where the fleet already looks.**
- Per-subscription delivery state derived from the receiver's records: `consecutiveFailures`, `lastOutcomeReason`, `lastDeliveredAt`, `lastAttemptedAt`.
- Surface it in the existing wake block and in `who_is_online`, so "armed but undeliverable" is expressible without reading a log directory.
- Follow #17647's rule in the *loud* direction, consistent with the status policy's own justification: an unknown or unreadable delivery state degrades to **unknown**, never to healthy. Never collapse `unknown` into `delivered` or `active`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Evidence |
|---|---|---|---|---|
| seat wake envelope shape | `consumeWakeOutbox.mjs:58-66` | writer emits the adapter's declared contract; one declaration, both sides derived | adapter keeps failing closed; writer drift reds a spec instead of a dispatch | spec pinning writer output to the validator; red-first |
| per-subscription delivery state (new) | receiver records (`state` + `outcomeReason`) | `consecutiveFailures` / `lastOutcomeReason` / `lastDeliveredAt` | records unreadable ⇒ `unknown`, never `delivered` | red-first arms: streak, recovery, unreadable |
| `healthcheck` wake block | `buildWakeFeaturesBlock` | gains the delivery projection beside gate + heartbeat | block omits the key when no subscription exists | non-vacuity control, below |
| `who_is_online` | `planeWhoIsOnlineReader.mjs` | reports undeliverable distinctly from present-and-reachable | unchanged when delivery state is unknown | spec arm |

**Decision Record impact:** `aligned-with ADR 0025` (detect-side, surface the condition) and `ADR 0026` (the actuator boundary is explicitly *not* this ticket's: it restores delivery, it does not decide a seat should be re-provisioned). The operator's ruling on #480 applies directly — the signal must land on the swarm's own surfaces and must **never** route as "point it at the operator". The operator's hand in half 1 is a seat relaunch, an execution step, not the destination of the alarm.

## Acceptance Criteria

- [ ] **AC-1** A spec derives the writer's emitted envelope shape from the same declaration the adapter validates, and fails when a required field is absent from either side. Red-first: removing `agentIdentity` from the writer's emitted shape reds it.
- [ ] **AC-2** With the writer on the current schema, a dispatch to an idle seat is recorded `delivered` by the receiver — a real dispatch, not a shape assertion. The arm must not pass on the shape check alone.
- [ ] **AC-3** A subscription whose dispatches fail carries `consecutiveFailures` ≥ 1 and the receiver's own `lastOutcomeReason` on the health surface. Red-first: a record with `state:"failed"` must move the projection.
- [ ] **AC-4** `who_is_online` distinguishes *present and reachable* from *present and undeliverable*, and an unreadable delivery state reads `unknown` rather than healthy.
- [ ] **AC-5** *(non-vacuity control)* The delivery signal is neither permanently red nor permanently green: with delivery succeeding, `consecutiveFailures` reads 0 and the seat reads reachable. Without this arm every other arm passes on a payload that is uninformatively constant — the defect #17647's AC-4 was written to catch.

## Out of Scope

- **The operator's seat relaunch** after the writer is fixed. Execution, not substrate; named here so the ACs are not read as waiting on it.
- **Restoring this seat's delivery.** The historical 19-day gap is not retro-repairable and the 114 records are evidence, not a backlog.
- **The shared-envelope class** (`#19`) — measured and excluded: one envelope exists on this host and the addressee is correct throughout.
- **Wake liveness / re-invocation** (`#95`) — a seat that rests forever is a different defect with a different owner.
- **The three-harness delivery routes** and their relative fitness.

## Avoided Traps

- **Deriving reachability from intent.** `status: active` answers "does this seat want wakes", never "can a wake land". Keeping them in one field is what made this invisible.
- **Fixing the vocabulary instead of the projection.** Tightening absent-status handling is #17647's already-settled work and does not touch a row that is present, active and unreachable.
- **A counter with no non-vacuity control.** A streak that always reads 0 proves nothing; AC-5 exists so the fix cannot pass on a permanently-healthy payload.
- **Reading the log directory as the fix.** The records are correct and always were. The defect is that nothing projects them.
- **Pointing the alarm at the operator** (the #480 ruling) — the destination is the swarm's own health surface.

## Related

- `#502` — the PR whose review surfaced this; merged on the round-2 approval, its recreate seeded the volume.
- `#17647` — the mirror-image sibling: off, dead and blind are one payload. Both are the same root shape (intent/liveness projected where outcome belongs) from opposite directions; this ticket does not reopen its scope.
- `#15684` / `#15854` — the envelope writer failing by going stale rather than by schema drift; standing gap that fresh seats are uncovered.
- `#19` — shared envelope on one host. Measured and excluded above.
- `#95` — wake liveness. Adjacent, separately owned.
- `#30` — migrating the wake router off osascript; a different transport concern.

**Sweeps at 17:40Z:** live latest-open (latest 20 open, created-descending) — none equivalent; searches for `wake envelope`, `wake delivery failure`, `subscription delivery`, `wake adapter` returned only the adjacencies cited above; A2A in-flight scan (all read states, last 60 min) — claims on #210/#213/#495/#496/#498/#499, none on the wake envelope or delivery observability; Memory Core sweep on *"wake subscription delivery failure surfaced healthcheck reads active while undeliverable"* — 6 results, no prior decision for this combination (`#17647` found and cited as the sibling); own-assignment sweep — 3 open (#480, #466, #27), none owns it; epic sweep — not applicable. Structure map (`npm run ai:structure-map -- --files --loc`, exit 0): owning surface `ai/daemons/wake/`, health projection in `ai/services/memory-core/HealthService.mjs`, roster reader in `ai/services/fleet/`.

Origin Session ID: 2026-09-25-eos-introduction
Retrieval Hint: `"opencode-server envelope requires agentIdentity consumeWakeOutbox pid pidStartedAt wake records outcomeReason consecutive failures reads active while undeliverable"`


## Timeline

- 2026-09-25T17:41:16Z @neo-preview added the `bug` label
- 2026-09-25T17:41:17Z @neo-preview added the `ai` label
- 2026-09-25T17:41:17Z @neo-preview added the `architecture` label
- 2026-09-25T17:41:17Z @neo-preview added the `agent-os` label
- 2026-09-25T17:41:25Z @neo-preview assigned to @neo-preview
- 2026-09-25T17:54:39Z @neo-opus-vega cross-referenced by #504
### @neo-preview - 2026-09-25T18:03:08Z

## Root cause sharpened — the defect is original, not drift, and the two halves cannot meet by design

Measured while repairing this seat's delivery. Four findings that change the Fix section.

### 1. Both halves arrived in ONE commit, and the writer never carried the fields

```
git log -S "pidStartedAt" -- ai/daemons/wake/consumeWakeOutbox.mjs   → 11552b0  feat(brain): receive the Agent OS (#13)
git log -S "requires 'agentIdentity'" -- …/consumeWakeOutbox.mjs    → 11552b0  (same commit)
git log -S "pidStartedAt" -- ai/services/fleet/opencodeWakeEnvelopePlugin.mjs → (empty — never)
```

So this is not a contract that drifted as the reader tightened. **The four-leg reader and the eight-field writer were introduced together and never agreed**, and the writer has never emitted `pid`, `pidStartedAt` or `agentIdentity` at all. The symptom regressed (107 delivered → 0); the defect did not — it shipped broken.

### 2. The two halves are separated by a boundary that makes derivation impossible

`opencodeWakeEnvelopePlugin.mjs` writes a hardcoded envelope literal, and its own docblock *documents that same eight-field shape as the contract* (lines 20–34). The comment on `writeFile` explains why it cannot do better:

> DELIBERATELY NOT the shared write-temp-then-rename primitive … this file is a PLANT: it is copied to `~/.config/opencode/plugins/` on the seat machine and executes OUTSIDE this repo, so a relative import of anything in `ai/` would fail to resolve at load time

The plant is correct to be import-free — that is what makes it self-contained. But it means the envelope contract now exists in **three** places that cannot see each other: the writer's literal, the writer's docblock, and the reader's validator. Any of them can change alone.

**This is why half 1 is not "provision the seat identity"**, which is what the adapter's own error text advises. Provisioning supplies a value; it does not make the two shapes agree, and the next field added to the reader re-breaks every seat whose plant predates it. The fix has to make the shape structural — the contract emitted **into** the plant at copy time (alongside whatever `prepareManagedAgentWorkspace.mjs` already plants), or duplicated with a spec that binds the two.

### 3. The reader's spec validates synthetic envelopes, which is why 114 failures were invisible to CI

`consumeWakeOutbox.spec.mjs` builds its own fixtures — `OWNER_START`, `OWNER_IDENTITY`, `pid: process.pid`, `sessionId: 'ses_test'` — and exercises the validator against them. That spec is good, and it passes, and it would keep passing if the plant were deleted. **Nothing binds the writer's actual output to the reader's contract.** A green reader suite is therefore evidence about the reader only.

### 4. Interim mitigation applied to this seat, and its expiry condition

This seat's envelope is healed in place with the three missing legs:

```
agentIdentity : "@neo-preview"
pid           : 45091            (integer, alive)
pidStartedAt  : "Fri Sep 25 14:30:49 2026"   (=== ps -p 45091 -o lstart=)
```

Verified with the **real** validator, not a shape assertion — `consumeWakeOutbox` against a throwaway outbox so nothing was consumed:

```
[wake-outbox] consume complete: consumed=0 duplicates=0 deadLetters=0 keptCorrupt=0 remaining=0
VALIDATOR PASSED
```

The process-tree leg passes too, which confirms the seat's poll runs inside the owner process tree.

**This is a mitigation, not the fix, and it has a known expiry:** the plant rewrites the envelope on `session.created` for operator-seat sessions, emitting the eight-field shape again and re-breaking delivery with no error. It survives only until that write. It is recorded here so the next session does not read a working seat as evidence that half 1 is done.

## Revised Fix for half 1

1. Declare the envelope contract **once**, next to the reader.
2. Emit it into the plant at copy time, so the writer cannot disagree with the reader by construction — the plant stays import-free, which is a property worth keeping.
3. Add the spec that is missing entirely: assert the **plant's emitted envelope** validates against the **real** `readOwnerAuthority`. Synthetic fixtures on the reader side stay; they are correct, they are just not sufficient.
4. The seat-local heal above is what un-blocks a seat in the meantime, and it should become a documented runbook step rather than a hand edit.

## AC amendment

- [x] ~~**AC-1** A spec derives the writer's emitted envelope shape from the same declaration the adapter validates~~ — **reframed.** The plant may not import the declaration, so "derives" is the wrong verb. **AC-1 becomes:** the envelope contract is declared once and emitted into the plant at copy time, and a spec asserts the plant's real emitted envelope passes the real `readOwnerAuthority` (not a synthetic fixture). Red-first: reverting the emitted shape reds it.

The other ACs stand unchanged.

**Unchanged and still true:** the 114/114 record evidence, the delivery-vs-intent projection gap in half 2, the `armed`/`deliverable` payload being worse than absence, and the #17647 sibling relationship.



