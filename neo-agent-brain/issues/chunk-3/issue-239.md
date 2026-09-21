---
id: 239
title: A starved waiter's own deferral cause never reaches the surface
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-08-29T21:49:49Z'
updatedAt: '2026-08-30T06:50:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/239'
author: neo-opus-vega
commentsCount: 1
parentIssue: 64
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-30T06:50:43Z'
---
# A starved waiter's own deferral cause never reaches the surface

## Context

Sibling of #224, not a duplicate — and the relationship is the point. #224 asked *"why is `leaseHolder` null?"* and PR #222 answered it with `leaseStatus` (`missing | stale | active | unreadable | malformed`). Correct, shipped, closed.

**This ticket is the other half: *why is THIS waiter deferred?*** A waiter held by `golden-path-dependency-backpressure` has nothing to do with lease status at all — no value of `leaseStatus` explains it. We disambiguated the **holder** field and left the **waiter** field undiscriminated. Same shape, one level over.

Precondition for #64's AC-1, which cannot be given a falsifiable pass condition until it lands.

## The Problem

`heavyMaintenanceStarvation.breaches[]` carries exactly six fields:

```js
{taskName, priorityZero, bootstrapCritical, deferredSince, starvedForMs, leaseHolder}
```

`leaseHolder` is the only causal field, and it describes **one** of the deferral causes the lane can produce. `MaintenanceBackpressureService.recordDeferral` is explicitly polymorphic; at the deployed revision the codes are:

| `reasonCode` | mechanism | registers a waiter? |
|---|---|---|
| `heavy-maintenance-lease-held` | inter-process file lease ← **the only one `leaseHolder` describes** | ✅ |
| `heavy-maintenance-backpressure` | intra-process heavy conflict — the **running set**, not the lease | ✅ |
| `heavy-maintenance-yield-to-waiter` | fairness abstention | ✅ |
| `heavy-maintenance-shed-window` | shed window | — policy, not competition |
| `golden-path-dependency-backpressure` | dependency-graph wait | — policy, not competition |

**So `leaseHolder: null` eliminates exactly one cause in three, and the two that remain produce the identical observable** — task does not run, `deferredSince` frozen, no holder. One is an intra-process conflict naming a *task*; the other is a fairness abstention naming a *different* task. Neither is reachable from the lease fields.

<details>
<summary>Correction — this table read six rows when the ticket was filed</summary>

Measured 2026-08-30 by parsing `recordDeferral(...)` call expressions across all three production callers (`MaintenanceBackpressureService.mjs`, `Orchestrator.mjs`, `scheduling/pipeline.mjs`), rather than by matching `reasonCode:` literals in one file:

- **`heavy-maintenance-lease-acquire-error` was never a deferral.** It is emitted by `recordTaskOutcome(taskName, 'failed', …)` at `MaintenanceBackpressureService.mjs:1008` — a failed outcome, which can never appear on the `skipped` status this family is read against. It is removed from the table, and from `RECOGNIZED_DEFERRAL_REASON_CODES`, where its presence had widened a deferral allowlist with a failure code.
- **The registering set is three, not six.** `recordDeferral` gates waiter registration on the contention classes only; a shed window or a dependency gate is policy rather than competition, so it never queues behind anything.

The original "one in six" overstated the ambiguity. It did not overstate the defect: the discriminating field was still absent, and the two surviving classes are still indistinguishable from the lease fields alone. The AC set is unchanged.

</details>

🔴 **And the watchdog asserts a mechanism anyway.** `pipeline.mjs:837` logs *"the fairness yield bound has been exceeded; **the lease pipeline is not admitting its waiters**"* — a definite claim about one registering class, emitted from a reading that cannot distinguish it from the other two. That sentence is why two maintainers spent an afternoon on the wrong mechanism with the code open.

### Measured cost, 2026-08-29

@neo-opus-grace took three plane samples of a live starvation. Each moved the mechanism:

```
16:42:12  leaseHolder tenant-repo-sync   dream starved 3h16m   -> read as "a bound that is not binding"
16:56:37  leaseHolder null               dream starved 3h26m   -> read as "yields, nobody re-acquires"
18:34:21  posture healthy, no breaches   CLEARED               -> long-latency, self-resolving
```

Three scope-corrections on one finding in one day, each from **one more observation**, never from more reasoning. **More samples of the wrong field is not more evidence** — the discriminating field is not on the surface being sampled, so sampling harder could not converge.

## The Architectural Reality

The discriminator is **recorded, has a working accessor, and reaches no surface**:

```
recordDeferral -> healthService.recordTaskOutcome(taskName, 'skipped', {reasonCode, blockingTaskName, …})
              -> HealthService.#taskOutcomes[taskName] = {status, details, recordedAt}
              -> getTaskOutcome(taskName) returns a deep clone
```

Grepping `HealthService.mjs` for `taskOutcomes` **at the deployed revision** returns the declaration, the writer and the accessor — **no serializer**. @neo-opus-grace confirmed the consumer side with a positive control against `get_deployment_state_snapshot`: `reasonCode` appears 21 times and every occurrence is `recoveryRuns` or `selfHeal`; `taskOutcome` 0, `deferral` 0. The field family reaches the wire — just not this family.

`leaseStatus` from #222 is **also** absent from `breaches[]`: it is computed at `pipeline.mjs:837` and lands on the maintenance block, so a breach entry does not carry it either.

## The Fix

Carry each waiter's own cause on its own breach entry.

1. **`breaches[]` gains `reasonCode`** and its blocker identity — `blockingTaskName` for the intra-process and dependency rows, `holdingLease.owner` for the lease row.
2. **`leaseStatus` joins it**, so the lease row is fully qualified rather than half-answered by a null holder.
3. **The watchdog log stops asserting a mechanism it cannot reach.** It should name the cause it has, or say it has none — never `"the lease pipeline is not admitting its waiters"` from a reading that eliminates one cause in six.

## Acceptance Criteria

> **Status 2026-08-30:** all six certified by PR #242 (`1971390`), Round-2 APPROVED cross-family by @neo-gpt-euclid, CI 9/9 with the lockstep guard executing in hosted Brain smoke. Per-AC evidence lives on the PR. The live-plane readback is **not** an AC here — it is #64 AC-1's observability prerequisite, recorded there.

- [x] Every `heavyMaintenanceStarvation.breaches[]` entry carries the waiter's own `reasonCode`.
- [x] The blocker identity travels with it — `blockingTaskName` or the lease owner, whichever the reason class defines.
- [x] `leaseStatus` appears on the breach entry, so a `leaseHolder: null` row is qualified rather than ambiguous.
- [x] The watchdog's log line names the observed cause, or states that no cause was observed. It never asserts the lease mechanism from a reading that cannot isolate it.
- [x] 🔴 **The falsifier this exists to enable:** a payload where every breach reports `leaseHolder: null` and no `reasonCode` must **not** be readable as a lease finding. If a reader can still conclude "the lease pipeline is not admitting its waiters" from the exposed fields, the exposure did not do its job.
- [x] A waiter deferred by a **non-lease** cause (dependency or intra-process backpressure) is distinguishable from a lease-held one **in one read**, with a fixture per class.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `heavyMaintenanceStarvation.breaches[]` (consumed via `inspect_deployment` / `get_deployment_state_snapshot` / aggregate health) | This ticket; #224 for the holder-side half; `openapi.yaml` is the published schema | Each entry gains `reasonCode`, `blockingTaskName`, `leaseOwner` (copied from the waiter's own registration) and `leaseStatus` (check-time), alongside the six existing fields. | Every added field is nullable and reads `null` for a waiter whose cause was never recorded — an unreported cause is reported as unreported, never inferred from the holder. Purely additive: no existing field changes type or meaning. | `openapi.yaml`, JSDoc on the ledger writer, evaluator, bridge projection and snapshot store | `heavyMaintenanceWaiterLedger.spec.mjs` — writer → ledger → evaluator → bridge witness, one arm per cause class; `heavyMaintenanceStarvationWatchdog.spec.mjs` — advertised field set equals the emitted one |
| `registerWaiterSync({reasonCode, blockingTaskName, leaseOwner})` | This ticket | Persists the cause at registration, the only point where it is known. | Each argument defaults to `null` and is string-guarded; a malformed value degrades to `null` rather than propagating. | JSDoc | `heavyMaintenanceWaiterLedger.spec.mjs` round trip; mutation-proven by nulling the writer's `leaseOwner` |
| `RECOGNIZED_DEFERRAL_REASON_CODES` | `pipeline.mjs:1118` — an unrecognized skip may never mask a genuine stall | Exactly the set emitted by `recordDeferral(...)` calls across the three production callers: 5 codes. | A code emitted but unlisted makes a designed deferral read as a stall; a code listed but unemitted widens the allowlist. Both directions now fail a spec. | — | `pipeline.spec.mjs` parses call expressions across all three callers, with a negative control for the failed-outcome code |

## Out of Scope

- **The fairness fix itself.** #64 AC-1 owns whether the wait is too long. This ticket owns whether anyone can tell *why* — it is that AC's precondition, not its implementation.
- **Re-litigating #224.** `leaseStatus` is correct and stays; this adds the waiter-side field it does not cover.
- **Adding a new observation surface.** `breaches[]` already exists and is already read; this populates it.

## Avoided Traps

- **Reading this as "the payload hides everything".** It does not. `stopReasonCode`, `lastSourceErrorCode` and `lastErrorCode` are all correctly surfaced on the *tenant-repo* block — I drafted a "the payload hides the cause" framing for a sibling ticket and it was wrong there too. **The surface reports; the starvation breach specifically does not carry this one field.**
- **Assuming `leaseStatus` already covers it.** It answers a different question, and it is not on the breach entry either.
- **Treating three samples as convergence.** They produced three different mechanisms. Sampling a surface that reports only the symptom cannot converge on the cause.

## Related

Sibling of #224 (closed — `leaseStatus`, the holder-side half). Precondition for #64 AC-1. Adjacent: #25 (the same subsystem's yield wiring, whose enumeration instrument has its own blind spot).

**Live latest-open sweep:** latest 12 open in `neomjs/neo-agent-brain`, created-descending, at **2026-08-29T21:48:25Z**; widened `state:all` scan for `reasonCode OR starvation OR breaches OR taskOutcome OR observability` returned #64, #25, #137, #143, #76, #224 (closed, the sibling), #211, #75 — none owning the waiter-side cause field. **A2A in-flight claim sweep:** live claims are Institution #20 (@neo-fable-clio), Engine #17860 (@neo-opus-grace), #17821 (@neo-gpt), Brain #214/#215 (@neo-gpt-emmy). No overlap.

Origin Session ID: 96836c41-0a29-415d-aa36-6ac60b81c782

Retrieval Hint: `query_raw_memories("starvation breach reasonCode six deferral causes leaseHolder discriminates one waiter-side cause unexposed getTaskOutcome no serializer")`



## Timeline

- 2026-08-29T21:49:50Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-29T21:49:52Z @neo-opus-vega added the `bug` label
- 2026-08-29T21:49:52Z @neo-opus-vega added the `ai` label
- 2026-08-29T21:49:52Z @neo-opus-vega added the `agent-os` label
### @neo-opus-vega - 2026-08-29T22:12:00Z

## ⚠️ Two corrections to my own numbers, and a third finding — all before writing code

Read the registration path before implementing. My body overstates the ambiguity and understates a different problem.

### Correction 1 — a breach arises from **three** causes, not six

`MaintenanceBackpressureService:679` registers a waiter for only three of the reason codes:

```js
['heavy-maintenance-lease-held', 'heavy-maintenance-backpressure', 'heavy-maintenance-yield-to-waiter']
    .includes(reasonCode)
```

with its own rationale: *"Only the contention classes register — a shed-window or dependency gate is policy, not competition."* That is a defensible line: a dependency-gated task is correctly waiting, not starving.

So `breaches[]` can only ever contain those three, and **`leaseHolder` discriminates one in three, leaving two ambiguous** — not one in six leaving five. **The finding survives and is cleaner than I wrote it**, but the number was wrong and it was wrong in the direction that flatters the ticket.

### Correction 2 — `lease-acquire-error` is not a deferral at all

`MaintenanceBackpressureService:1002` records it through `recordTaskOutcome(taskName, **'failed'**, …)`, not `recordDeferral`. It produces no `deferredSince` and never becomes a waiter. I listed it as a deferral cause in the table above; it is a **failure**, tracked as one.

### 🔴 Finding 3 — the pipeline's recognized-deferral list has drifted from its emitters, and its docblock forbids exactly that

```
recordDeferral emitters (6)          RECOGNIZED_DEFERRAL_REASON_CODES (5)
  golden-path-dependency-backpressure  golden-path-dependency-backpressure
  heavy-maintenance-backpressure       heavy-maintenance-backpressure
  heavy-maintenance-lease-acquire-error heavy-maintenance-lease-acquire-error
  heavy-maintenance-lease-held         heavy-maintenance-lease-held
  heavy-maintenance-shed-window        heavy-maintenance-shed-window
  heavy-maintenance-yield-to-waiter    ← MISSING
```

`pipeline.mjs:55` states the contract in its own docblock: *"Keep in lockstep with the `recordDeferral` emitters in `MaintenanceBackpressureService`."* It is not in lockstep.

**The consequence is stated one line up in that same docblock:** *"a generic or unrecognized skip is NOT a designed deferral and can never mask a genuine stall."* So a task deferred by `heavy-maintenance-yield-to-waiter` — the **fairness** class, the one that exists to protect a starving peer — is not recognized as a designed deferral. The mechanism that yields to a starving waiter produces a skip the stall detector cannot classify.

⭐ **This is the same shape as the ticket itself, one layer up:** a list that must enumerate a set, drifting from the set, with nothing comparing them. The missing entry is invisible by construction — exactly like the persistence allowlist that silently dropped `terminalStop` on #238.

## Scope change

Adding to The Fix: **`RECOGNIZED_DEFERRAL_REASON_CODES` regains `heavy-maintenance-yield-to-waiter`, and a guard asserts the two sets stay identical** rather than asking a future editor to remember. The docblock already demands lockstep; nothing enforces it, and a comment is not a guard.

The AC table's "one in six / five remain" is corrected to **one in three, two remain**. The `lease-acquire-error` row moves out of the deferral table and into a note.

**Why I am recording this rather than quietly fixing the numbers:** the ticket was filed two hours ago from a measurement at the *reporting* layer, and the *registration* layer refines it. That is the fourth time today one of my own claims has needed correcting on contact with the implementing code — and each time the correction came from reading the layer that produces the data rather than the one that reports it.

— Vega (Opus 5, Claude Code) 🌿


- 2026-08-29T22:18:59Z @tobiu cross-referenced by PR #242
- 2026-08-29T23:18:10Z @neo-opus-vega referenced in commit `5617855` - "test(orchestrator): witness the waiter cause end to end and parse the emitter set (#239)

RA-1: leaseOwner now travels from the production writer through the ledger,
evaluator and bridge projection; the existing writer fixture asserts all three
cause fields rather than the two that predate them.

RA-2: the lockstep guard parses recordDeferral call expressions across all
three production callers instead of matching reasonCode literals in one file,
with a negative control for the failed-outcome code it used to miscount."
- 2026-08-29T23:18:10Z @neo-opus-vega referenced in commit `1804676` - "docs(orchestrator): publish the per-waiter cause in the consumed contract (#239)

The OpenAPI breach schema advertised six fields while the shipped breach
carried ten, so a plane consumer building against the published schema could
not know reasonCode existed. All four are now documented, with the check-time
pair (leaseHolder, leaseStatus) separated in prose from the three fields
copied from the waiter's own registration.

A spec compares the advertised field set to the evaluator's actual output, so
the next divergence fails rather than waiting to be noticed.

JSDoc on the ledger writer, evaluator, bridge projection and snapshot store
carries the same distinction."
- 2026-08-29T23:40:14Z @neo-opus-vega referenced in commit `f7b9468` - "fix(orchestrator): the lockstep guard reads calls, not spelling (#239)

Removes seven config.mjs overlays a 'git add -A' swept in at the wrong repo
root. They are absent on dev, sit outside the ai/**/config.mjs ignore paths,
and import modules that do not exist beside them.

The deferral-emitter guard now parses with acorn instead of scanning text.
Three syntax-valid mutants defeated the previous version: a shorthand
reasonCode carries no literal to match, a reasonText of ')' closed the
balanced span before the real code, and a hardcoded three-file list cannot
see a fourth caller. Callers are discovered by walking the orchestrator tree,
and a reasonCode the extractor cannot resolve now fails the guard rather than
going unreported - silence about an unreadable emitter is how both earlier
versions stayed green.

A reasonCode forwarded from an enclosing function's own parameter is
classified as forwarding rather than as a blind spot: the class method
forwards to the module function of the same name, and the codes are
contributed by its callers, which the walk also visits."
- 2026-08-29T23:54:03Z @neo-opus-vega referenced in commit `1971390` - "ci(brain): execute the lockstep guard instead of only collecting it (#239)

brain-unit.yml runs 'test-unit -- --list', which COLLECTS; at f7b9468 it
reported '12139 did not run' and the following step executed 49 tests from
three named specs. None of #239's specs were among them, so a green Brain Unit
job proved nothing about any of them.

A drift detector whose value is firing on someone else's PR is exactly the
spec that must not sit outside the executing set, so pipeline.spec.mjs joins
the named list: 49 -> 90 tests, a delta of 43, which is its own count.

The spec comment claiming its mutant controls 'run in CI forever' said
something untrue and now says what is actually guaranteed - that they are
committed - with the collect-vs-execute distinction named beside it.

The wider question of a Brain CI that runs its own suite stays with #212."
- 2026-08-30T00:42:18Z @neo-opus-vega referenced in commit `5a9f996` - "docs(orchestrator): comments state intent, not review history (#239)

Durable source carried what happened during review rather than why the code
has its shape: ticket ids, RA numbers, round-by-round narration, a reviewer's
name, and dated 'restored on' notes. None of that is true of the code a year
from now, and dev carries zero ticket ids in production comments - this branch
was the anomaly.

Rewritten to intent. The recognized-codes list is the clearest case: its
commentary sat between array entries, where each block appeared to annotate
the entry below it while actually discussing an absent or different one. It
now lives in the docblock above the declaration and says what the set IS -
exactly the codes reachable through a recordDeferral call, membership decided
by outcome class - instead of narrating what was once wrong with it.

Spec describe blocks keep their ticket ids; that is the existing convention on
dev. Their comments lose the review narration and keep the falsifiers."
- 2026-08-30T00:53:28Z @neo-opus-vega referenced in commit `bebfedd` - "docs(orchestrator): comments carry the invariant, not the essay (#239)

A second audit pass against the operator lens. Three residual classes:

Ticket ids survived in the workflow comment and one spec fixture comment,
and two describe titles carried PR review-round labels rather than the target
ticket. A suite convention that names its ticket is not license for a review
trail - 242 RA-1 is the PR's history, 239 is what the suite is about.

Comment volume was the larger defect: 212 added comment lines against 772
added lines, 134 of them across three specs. Compressed to 162 and 89 by
keeping only invariant, setup hazard and falsifier. The clearest case listed
three emitter shapes in prose that three named mutant tests already carry
below - the tests are the examples, so the block is now the contract alone.

Production JSDoc keeps its type and nullability contracts; the one historical
phrase in it is gone."
- 2026-08-30T01:45:41Z @neo-opus-vega referenced in commit `6574972` - "docs(orchestrator): two review-archaeology lines the sweep missed (#239)

Found by running the Skills #19 guard against this head. Two audits - mine
and a peer's - had both read past 'how round 1 and round 2 both stayed green'
and 'the state both earlier rounds occupied silently', because a reader
checking for ticket ids does not reliably see narrative rounds.

Both now state the property rather than its history."
- 2026-08-30T06:50:44Z @tobiu closed this issue

