---
id: 1074
title: Stop fleet leaves a seat unstartable when its Codex Desktop helper cleanup fails
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gemini-pro
createdAt: '2026-10-16T23:47:19Z'
updatedAt: '2026-10-17T18:46:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1074'
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
closedAt: '2026-10-17T18:46:26Z'
---
# Stop fleet leaves a seat unstartable when its Codex Desktop helper cleanup fails

## Context

Operator-observed on 2026-10-16 (Tobi, reported 2026-10-17 ~01:3x Berlin; defect-note `MESSAGE:33a39d07-2193-4aed-8682-3f4a316e27e9`, fingerprint `ac1a718e2937dab`). **Observation:** with every seat idle for more than an hour and a Gemini harness update pending, the operator pressed Stop fleet. The Fleet Manager closed every harness. The fleet button then read `Start fleet · 7` of nine active seats, two cards kept their Stop verb, and nothing in the Fleet Manager could start those two seats; quitting and relaunching the Fleet Manager freed them. **Inference, marked as such:** the operator does not remember which two cards; the five Claude seats were among the seven that started afterwards (they showed the stale-issuer line of the relaunch, Ada's note `MESSAGE:f2650588`), so the two were most likely Codex Desktop seats — the only seat class whose stop has a second phase. Stop fleet followed by Start fleet had worked repeatedly on prior days; this is intermittent. The Fleet Manager's `main.log` carries no stop lines for the window (only the SIGINT of the quit at 22:39:55Z and the relaunch at 22:40:06Z), so the two seats' recorded reasons are not in hand; the mechanism below is traced from source and fits every observed symptom.

## The Problem

A seat whose stop ended in a failed or ambiguous helper cleanup is neither up nor down to the Fleet Manager: the roster reads it as not `off`, the fleet button counts it as "already up", the card offers Stop (a retry of the cleanup) and refuses Start, and the only exit is a Fleet Manager quit — which costs every other seat its launch admission (`f2650588`). The operator sees a count that does not add up, two Stop buttons on closed harnesses, and no word why.

## The Architectural Reality

- `ai/services/fleet/FleetLifecycleService.mjs`: on the tracked main's exit (`:2500–2528`), a record without `electronProfile` becomes `stopped` at once; a Codex Desktop record goes through `finalizeCodexDesktopHelpers` (`:2538–2561`): `state: 'stopping'` until the exact-profile helper cleanup proves zero residuals, then `stopped`; on any thrown error `state: 'failed'`, `cleanupUnresolved: true`, `failureReason: 'Codex Desktop helper cleanup failed: …'`. The method's own JSDoc: *"A transient ambiguous scan remains a durable failed state."*
- `ai/services/fleet/manageCodexDesktopRuntime.mjs` `cleanupCodexDesktopCrashpad` (`:254–312`): *"Ambiguity at any phase fails"* — a profile-matching helper row classified `ambiguous` (`classifyCodexDesktopCrashpadProcesses` `:124–166`: `database-identity-unavailable`, `multiple-profile-database-arguments`, `executable-identity-unavailable`, `profile-match-with-foreign-executable`, `process-birth-token-unavailable`), an unavailable process table (`:197`) or a helper surviving SIGKILL (`:311`) throws. The fail-closed proof is deliberate and right; the dead end after it is the defect.
- `FleetLifecycleService#start` (`:765–766`) refuses to spawn while `priorRecord.state === 'stopping' || priorRecord.cleanupUnresolved` ("refusing to spawn agent … while Codex Desktop helper ownership is … unresolved after a lifecycle failure"). `stop()` (`:1404–1431`) still accepts such a record and re-runs the finalizer — so the card keeps a Stop verb whose only effect is another scan.
- The roster projection: a `failed` / `stopping` record is not session state `off` (Institution `apps/agentos/util/SourceHealth.mjs:247–320`), so `FleetStartPlan#partitionFleetStart` (Institution `apps/agentos/util/FleetStartPlan.mjs:101–102`) excludes it as `already up — session state '…'` — the `Start fleet · 7`. The card's control status line (`apps/agentos/CARD-CONTRACT.md`, control status) words `pendingAction` / `controlReason` and launch admission, not `cleanupUnresolved`; the reason never reaches the operator.
- Lifecycle records live in the Fleet process; a quit drops them and the relaunch re-adopts from the persisted journal — the operator's workaround, and the cause of `f2650588`.

## The Fix

Give the unresolved state an exit inside the Fleet Manager and a name on the card:

1. **Start re-proves instead of refusing.** `start()` on a record with `cleanupUnresolved` re-runs the exact-profile helper proof once (the same `finalizeCodexDesktopHelpers` the Stop retry runs) and proceeds to spawn when the process table shows zero owned or ambiguous helpers; it refuses only while a helper that the profile owns is still alive or the table is unreadable, and then says which.
2. **The reason travels.** `lifecycleStatus` / the roster row carries `cleanupUnresolved` with `failureReason` so the Institution's card can print `helper cleanup unresolved: <reason> — retry` on its control status line (CARD-CONTRACT, a new `controlReason` kind beside `timeout`), and `FleetStartPlan` excludes the seat with that reason instead of `already up`.
3. **No proof is relaxed.** The zero-residual requirement stays; an ambiguous row still fails the stop. What changes is that failure is a state the operator can act on from the roster, not a dead end.

The Fleet steward decides the primitive; the Institution half (card words, plan reason) is a sibling leaf under Institution #12 once this ticket's wire shape is fixed.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService#start(id)` result for a `cleanupUnresolved` record | `FleetLifecycleService.mjs:765` (today: throws while unresolved) | Re-proves the helper cleanup once; spawns on zero owned/ambiguous helpers; refuses with `failureReason` naming the live helper or the unreadable table | A thrown refusal with the reason words, never a silent no-op | JSDoc on `start`; `learn/` fleet lifecycle page if one names the stop phases | Unit arm AC-1, AC-2 |
| `FleetLifecycleService#lifecycleStatus(id)` row (`:1540`) | the existing row shape (`state`, `failureReason`, `cleanupUnresolved`) | Unchanged fields; `cleanupUnresolved` + `failureReason` guaranteed present for a failed helper cleanup | — | the Institution reads it (`SourceHealth`); shipped also on the `FleetManager#fleetRuntimeStatus` row (`cleanupUnresolved` when true, `failureReason`) and the `createFleetCockpitStatus` row (both always present) — Brain PR #1082 | Unit arm AC-2 |
| Institution card control status + `FleetStartPlan` exclusion reason | CARD-CONTRACT.md control status; `FleetStartPlan.mjs:101` | `helper cleanup unresolved: <reason> — retry` on the card; plan reason names it, not "already up" | the existing `⚠ <kind>: <reason>` branch | CARD-CONTRACT.md | Institution #725, AC-3 |

Decision Record impact: none (a lifecycle repair inside the existing stop/start contract; no ADR names the helper-cleanup phase).

## Acceptance Criteria

- [ ] AC-1 Unit (Brain): with a `codexDesktopCleanupFn` that throws an ambiguous-identity error once and then returns clean, `stop()` resolves `{success: false, state: 'failed', cleanupUnresolved: true}` and a subsequent `start()` re-proves and spawns — no Fleet restart in between.
- [ ] AC-2 Unit (Brain): with a cleanup that keeps failing, `start()` refuses with a reason naming the live helper (or the unreadable process table), and `lifecycleStatus(id)` carries `cleanupUnresolved: true` + that `failureReason`.
- [ ] AC-3 (Institution sibling leaf, filed on this ticket's wire shape): the card prints the reason on its control status line and `Start fleet · n` excludes the seat with that reason; the `partitionFleetStart` unit arm asserts the words. **[L4-deferred — Institution #725 owns it]**
- [ ] AC-4 (post-merge, installed) On the next Institution #12 candidate: one Stop fleet → Start fleet cycle with all seats, receipt on the Row 1 walk (Fri Oct 23); a seat left unresolved shows its reason and starts from the roster. **[L4-deferred — operator handoff needed: the Row 1 walk, 2026-10-23]**

## Out of Scope

The stale-issuer refusal after a Fleet Manager relaunch (`f2650588`, its own ticket); the Fleet Manager quit not stopping seats (Vega's 2026-10-09 note); Start-fleet pacing and admission proofs under load (#964); the Codex Desktop helper classification rules themselves.

## Avoided Traps

Relaxing the zero-residual proof or treating an ambiguous helper as clean (the fail-closed scan is the custody guarantee; the defect is the missing exit, not the proof); inferring `off` from a missing child process (the record's custody is the point of the phase); a Stop verb that silently retries forever without a word; a quotation of the operator's report as the root cause (the report is the symptom; the mechanism is traced at the lines above and stays a hypothesis at the seat level until a seat's recorded `failureReason` is read).

## Related

Institution #12 (the installed-candidate epic; AC-4's walk), Institution #725 (AC-3's leaf), Brain PR #1082 (the Brain half, approved at 6664e85b), Institution #618 (the fleet button reads its plan), Institution #690 (header counts after lifecycle changes), neo #17305 (runtime status `stopped` for seats never launched), Brain #964 (launch admission under load), defect-notes `33a39d07` (this) and `f2650588` (the stale issuer, Ada).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-16 23:45:13Z; nearest #936 (wake activation must raise the addressed Codex instance) is a different Codex seam; no equivalent. Exact sweep (`stop fleet`, `cleanupUnresolved OR Crashpad OR "helper cleanup"`): Institution #618 and #690 (closed, the button and the counts), neo #17305 (closed) — none on the unresolved-stop dead end. A2A in-flight claim sweep at 23:45Z: none on this scope (Ada's `f2650588` is the adjacent issuer defect). MC sweep: "Stop fleet two seats stayed running cards kept the Stop button Start fleet count excluded seats could not start without quitting the Fleet Manager" — 6 results: Mnemosyne's 2026-10-09 answer on the start plan (455a259b), the #618 filing (5436edee), Vega's 2026-10-09 census that an FM quit does not stop seats (ce7f6427), my #964 record; no prior decision on the stop finalizer's dead end. Own-assignment sweep: 5 open in the Brain (#977 #850 #50 #51 #53), 3 in the Institution (#507 #505 #351), none overlapping. Structure map: ran at Brain `dev` (exit 0); no new module — the owners are `ai/services/fleet/FleetLifecycleService.mjs` and `manageCodexDesktopRuntime.mjs`.

unowned-rationale: filed from the operator's report by the design seat; the Fleet lifecycle steward self-selects (the stop/start phases are Grace's and Vega's prior ground; Ada holds the sibling issuer defect).

Origin Session ID: 8daba841-613f-486f-8e8d-c0ba5a782d2a

Retrieval Hint: "Stop fleet Codex Desktop helper cleanup unresolved start refuses card keeps Stop Start fleet 7 of 9 FM quit"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 8daba841-613f-486f-8e8d-c0ba5a782d2a


## Timeline

- 2026-10-16T23:47:21Z @neo-fable-clio added the `bug` label
- 2026-10-16T23:47:21Z @neo-fable-clio added the `ai` label
- 2026-10-16T23:47:21Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T16:03:00Z @neo-fable-clio cross-referenced by #1077
- 2026-10-17T16:11:54Z @neo-fable-clio cross-referenced by #1078
- 2026-10-17T16:50:26Z @neo-fable-clio cross-referenced by PR #1079
- 2026-10-17T17:06:18Z @neo-fable-clio cross-referenced by PR #1080
- 2026-10-17T17:25:41Z @neo-fable-clio cross-referenced by #1081
- 2026-10-17T17:48:27Z @neo-gemini-pro assigned to @neo-gemini-pro
- 2026-10-17T17:59:07Z @neo-gemini-pro cross-referenced by PR #1082
- 2026-10-17T18:43:00Z @neo-fable cross-referenced by #727
- 2026-10-17T18:46:26Z @tobiu referenced in commit `afae1ef` - "fix(fleet): reprove Codex Desktop helper cleanup on start after stop failure (#1074) (#1082)"
- 2026-10-17T18:46:27Z @tobiu closed this issue
- 2026-10-17T19:15:22Z @neo-gemini-pro cross-referenced by PR #728

