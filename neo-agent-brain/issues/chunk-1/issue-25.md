---
id: 25
title: Three heavy-lease holders never consult the lease's hold bound
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-08-19T09:08:46Z'
updatedAt: '2026-11-02T18:36:09Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/25'
author: neo-opus-vega
commentsCount: 2
parentIssue: null
subIssues:
  - '[x] 17398 The container tenant sweep never consults the lease it holds'
  - '[x] 17414 A lease-caused yield cannot release the lease, because the sweep never learns which cause stopped it'
subIssuesCompleted: 2
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Three heavy-lease holders never consult the lease's hold bound

**Rewritten 2026-11-02 against Brain `dev` `92a7ec44`** for the #1438 backlog sweep. The original body is in this issue's edit history: it was filed 2026-08-19 at operator direction, with its falsified first premise struck in place and the AC-1 enumeration. Vega's 2026-08-29 comment dates the fix.

## Shipped

The original headline, `tenant-repo-sync` holding the shared heavy-maintenance lease without consulting its hold bound, is fixed by neomjs/neo#17398 (PR neomjs/neo#17424, merged 2026-08-21, with neomjs/neo#17456). On `dev`:
- the container one-shot passes `leaseYieldVoter: createLeaseYieldVoter(acquisition)` (`ai/scripts/maintenance/syncTenantRepos.mjs:214`);
- the pipeline injects the voter into in-process tasks (`MaintenanceBackpressureService.mjs:1133`);
- `TenantRepoSyncService` resolves its yield cause from that voter and its own slice budget (`:2846`).

## What remains

Three paths take the shared lease through `withHeavyMaintenanceLease` and consult no hold bound. None references `shouldYield`, `leaseYieldVoter`, `createLeaseYieldVoter`, `maxActiveHoldMs` or `shouldYieldHeavyMaintenanceLease`:
- `ai/scripts/lifecycle/backfill-memory-summaries.mjs:77` (`memory-summary-backfill`)
- `ai/scripts/maintenance/syncGithubWorkflow.mjs:234` (owner `syncGithubWorkflow`)
- `ai/scripts/maintenance/ingestTenant.mjs:201` (the tenant ingest CLI)

Each can keep the exclusive heavy slot past `maxActiveHoldMs`, which defaults to 30 minutes (`ai/configBase.mjs:2075`), while every starved heavy peer waits. The search's boundary is the three files themselves, with `syncKnowledgeBase.mjs` as its positive control. A callee that consults the bound under another name would not show up, as Vega's 2026-08-29 comment warns about the original table.

## Acceptance Criteria

- [ ] Each of the three paths consults the lease voter at a checkpoint, or this ticket records the measurement that shows it cannot outlive the bound.
- [ ] For each path that yields, a fixture whose work would otherwise run past `maxActiveHoldMs` shows the hand-back, and a starved peer is observed to run. A task that finishes quickly is not coverage.
- [ ] A heavy-lease holder that consults nothing is detectable without an incident.
- [ ] A mutation that removes a new consultation fails its own fixture and only that one.

Related: neomjs/neo#17398, neomjs/neo#17424, #64 (which deferral cause ran the clock).

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-08-19T09:08:47Z @neo-opus-vega added the `bug` label
- 2026-08-19T09:08:47Z @neo-opus-vega added the `ai` label
- 2026-08-19T09:08:47Z @neo-opus-vega added the `agent-os` label
- 2026-08-19T09:08:59Z @neo-opus-vega cross-referenced by #17379
- 2026-08-19T09:23:05Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-08-19T20:11:18Z @neo-opus-vega cross-referenced by #17398
- 2026-08-19T20:12:31Z @neo-opus-vega cross-referenced by PR #17399
- 2026-08-20T11:55:18Z @neo-opus-vega cross-referenced by #17414
- 2026-08-20T16:31:57Z @neo-opus-vega cross-referenced by PR #17424
- 2026-08-21T00:31:56Z @tobiu referenced in commit `5f36f95` - "fix(ai): the tenant sweep consults the shared lease bound, not only its own slice budget (#17380)

The container path takes the SHARED heavy-maintenance lease and never asks how
long it has held it. `ai/scripts/maintenance/syncTenantRepos.mjs` acquires that
lease as owner `tenant-repo-sync` and contains zero yield consultations, while
the sweep beneath it bounds only each repo's share of the sweep. A sweep over N
repos therefore honours every per-repo slice budget and still occupies the
exclusive-heavy slot for roughly N x sliceBudgetMs -- the slice budget rotates
WITHIN a sweep rather than ending it.

Measured on a live plane: five waiters starved between 4 and 34 hours, every
breach naming that owner, while no budget the holder could see was ever broken.

The two bounds answer different questions and neither subsumes the other -- the
slice budget is fairness BETWEEN REPOS, the lease bound is fairness BETWEEN
TASKS -- so they compose with OR via a new pure composeYieldPredicates. A caller
holding no outer lease passes nothing and keeps today's behaviour exactly.

The owner string is what identifies the holder: the in-process service stamps
tenant-repo-sync:scheduler while this CLI stamps the bare tenant-repo-sync,
which is the form the starvation watchdog reports.

Not claimed: which shape the live instance has. starvedForMs measures the
waiter's wait, so a continuously re-acquiring holder and one long hold read
identically. A path that takes the shared slot and consults no fairness bound is
a defect either way.

3850 orchestrator + scripts unit tests pass. Two exact-shape option pins caught
the new key and were updated rather than loosened; one now asserts the vote is
threaded on the real dispatch path."
- 2026-08-21T10:29:48Z @neo-gpt-euclid cross-referenced by PR #17456
### @neo-opus-vega - 2026-08-24T20:33:32Z

## AC-1 discharged — the enumeration, with its boundary, and three heavy tasks that consult nothing

Measured at `origin/dev@870752f08b`. AC-2/3/4 landed via PRs neomjs/neo#17456 and neomjs/neo#17424; what follows is the enumeration AC-1 asks for and it names AC-5's subject concretely.

### The two heavy enumerations AGREE — and that was not checked before

`TASK_REGISTRY` carries `maintenanceClass: 'heavy'` per descriptor; `MaintenanceBackpressureService` exports `DEFAULT_HEAVY_MAINTENANCE_TASK_NAMES`. **Two independent lists of "heavy", either of which a new task could join alone.** Compared programmatically: both 11, zero symmetric difference. No drift today — but nothing detects drift tomorrow, and that is half of AC-5.

### Heavy tasks partitioned by `executionKind` — the boundary is the partition

| `executionKind` | tasks | yield relevance |
|---|---|---|
| `supervised-child-process` (7) | `backup`, `core-corpus-projection`, `graphlog-compaction`, `kbSync`, `memory-summary-backfill`, `summary`, `temporal-summary` | run as child processes; a consultation, if any, happens **inside the child** — `kbSync` does exactly that (`syncKnowledgeBase.mjs:128`) |
| `service-runner` (2) | `primary-dev-sync`, **`tenant-repo-sync`** | in-process; needs its own predicate |
| `in-process-async` (2) | `dream`, `message-concept-harvest` | in-process; needs its own predicate |

**Boundary, stated as the AC requires.** Searched: `ai/**` for `shouldYield` / `resolveYieldCause` / `buildLeaseYieldPredicate`, excluding tests — nine production files, so the search reaches (positive control non-empty). NOT searched: whether a child process consults a yield after fork beyond the one case above, and any consultation expressed under a different name than those three.

### The finding: there is no dispatch-level yield provision, so three of four in-process heavy tasks consult nothing

`pipeline.mjs` — which owns the dispatch, `executeServiceRunnerCandidate` (`:442`) and `executeInProcessCandidate` (`:489`) — contains **zero** references to any yield predicate. So the dispatch hands nothing to any runner, and every in-process heavy task must construct its own.

Exactly one of the four does. `TenantRepoSyncService.mjs:2528` builds `shouldYield: () => resolveYieldCause() !== null` internally, which is what PR neomjs/neo#17456 landed.

| in-process heavy task | consults a yield? |
|---|---|
| `tenant-repo-sync` | **yes** — `TenantRepoSyncService.mjs:2528` |
| `primary-dev-sync` | **no** |
| `dream` | **no** — `DreamService.mjs` has zero yield references |
| `message-concept-harvest` | **no** |

`dream` is the one that matters most: the Dream Pipeline is long-running by design, so it can hold the heavy-maintenance slot past `maxActiveHoldMs` and starve peers — the exact incident class this ticket was filed about, still live, on the task least likely to finish quickly.

### What this leaves

AC-5 is now a named gap rather than a suspicion: **three in-process heavy tasks consult nothing, and nothing detects that.** Two candidate shapes, recorded rather than chosen:

1. **Dispatch-level provision** — `pipeline.mjs` supplies a predicate to in-process runners once, so a new heavy task inherits it instead of remembering to build one. Falsifier: if the lease context genuinely is not resolvable at dispatch time, this cannot work and each runner must own it.
2. **Per-task wiring plus a detector** — three small changes, plus a spec pinning that the two heavy enumerations agree AND that every in-process heavy task consults something, with the current gap as an explicit baseline carrying a retirement condition. Cheaper per step, and it leaves the next heavy task still able to forget.

No new ticket filed for either, per the 4:1 directive: this belongs to neomjs/neo-agent-brain#25 and stays here.

Measured by @neo-opus-vega (Fable 5, Claude Code) · origin session `cad88c79-073f-4816-aaa7-e779224f2af3`


- 2026-08-26T14:59:46Z @tobiu added the `bug` label
- 2026-08-26T14:59:46Z @tobiu added the `ai` label
- 2026-08-26T14:59:47Z @tobiu added the `agent-os` label
- 2026-08-28T15:37:05Z @neo-opus-vega unassigned from @neo-opus-vega
### @neo-opus-vega - 2026-08-29T18:41:00Z

## The headline is already fixed — it shipped 2026-08-21 under an Engine ticket, so this one never closed

This was filed at operator direction as *"the item blocking the next deployment."* It is not blocking anything: **the tenant-repo-sync half landed eight days ago.**

### Dated, not inferred

```
2026-08-19T09:08:46Z   #25 filed
                       TenantRepoSyncService.mjs @ ee7ba362f1 : leaseYieldVoter × 0   <- correct when written
2026-08-21T08:03:15Z   neomjs/neo PR #17424 MERGED
                       "feat(ai): both outer lease holders vote, and the vote reaches the sweep (#17398)"
2026-08-21T08:03:16Z   neomjs/neo#17398 CLOSED COMPLETED
                       "The container tenant sweep never consults the lease it holds"
```

`neomjs/neo#17398` is **this ticket's headline in different words**, filed and closed independently in the Engine while this Brain ticket stayed open. Verified in the Engine's history with `git log -S`, not from the commit subject alone.

**And the fix is on the plane.** At the deployed revision `467fd122f3`, `TenantRepoSyncService.mjs` carries `leaseYieldVoter` ×4. The sweep consults both bounds today: `createYieldCauseResolver([leaseYieldVoter, {cause: YIELD_CAUSE_SLICE, …}])`, with the lease voter built by `createLeaseYieldVoter` from `AiConfig.orchestrator.heavyMaintenance.maxActiveHoldMs` and threaded via `MaintenanceBackpressureService:1051 → pipeline.mjs:464`.

### ⚠️ Why the table's own instrument could not have caught the fix

The AC-1 table scores each holder by grepping the file for `maxActiveHoldMs`. `TenantRepoSyncService.mjs` still reads **0** for that key at HEAD — and honours the bound perfectly. The value is **injected** by the boundary rather than read locally, which is the reactive-provider pattern ADR-0019 requires.

So **the instrument measures local config reads, not bound consultation.** Its control (`kbSync`) passed only because that holder happens to read the leaf itself. A grep for a config key cannot see a dependency-injected bound, and re-running that table today would still report `tenant-repo-sync: no` against code that demonstrably votes. Worth fixing before the residual below is re-measured with it — otherwise the same instrument will mis-score the same way.

### What actually survives — and it is the other three rows

Re-measured at Brain HEAD just now:

| holder | `leaseYieldVoter` | `shouldYield` | consults the lease bound? |
|---|---:|---:|---|
| `TenantRepoSyncService.mjs` (`tenant-repo-sync`) | **4** | 1 | ✅ **fixed by #17398/#17424** |
| `backfill-memory-summaries.mjs` | 0 | 0 | ❌ still no |
| `syncGithubWorkflow.mjs` | 0 | 0 | ❌ still no |
| `ingestTenant.mjs` | 0 | 0 | ❌ still no |

### Recommended disposition — not taken unilaterally, since this was operator-directed

**Re-scope to the three remaining holders and strike the tenant-repo-sync framing**, or close as superseded by `neomjs/neo#17398` and file a successor naming those three. Either is defensible; what is not defensible is leaving an operator-directed *"blocking the next deployment"* ticket standing on a premise that shipped eight days ago. I have not touched its state — it is unassigned and was filed at @tobiu's direction, so the call is his or its author's.

**Cross-repo note, because this will recur:** the fix landed under an **Engine** number while the ticket lived in **Brain**. Nothing links them, and nothing closed the Brain row. Post-split, a Brain ticket whose subject still lives in the Engine tree can be resolved without any signal reaching it — this is the first instance I have measured, and it will not be the last.

**Adjacent, filed today:** neomjs/neo-agent-brain#64's AC-1 was re-aimed three times in one day against this same subsystem, and the reason is directly relevant here — `heavyMaintenanceStarvation.leaseHolder` discriminates **one deferral cause in six**, so no surface either maintainer could reach can say which cause ran the clock. The condition is real (3h26m against a wired 30-minute bound) and **self-resolves**. See neomjs/neo-agent-brain#64 for the six-code table and the observability gap.

— Vega (Opus 5, Claude Code) 🌿


- 2026-08-29T21:49:51Z @neo-opus-vega cross-referenced by #239
- 2026-09-23T12:33:27Z @neo-opus-vega cross-referenced by #430
- 2026-09-25T10:04:08Z @neo-opus-vega cross-referenced by #64
- 2026-09-25T15:43:36Z @neo-opus-ada cross-referenced by #495
- 2026-10-27T22:56:56Z @neo-opus-ada cross-referenced by #1174
- 2026-11-02T18:14:10Z @neo-opus-vega cross-referenced by #15000
- 2026-11-02T18:19:26Z @neo-opus-vega cross-referenced by #1438
- 2026-11-02T18:36:09Z @neo-opus-ada changed title from **The cooperative yield is wired per task, and tenant-repo-sync was never wired into it** to **Three heavy-lease holders never consult the lease's hold bound**

