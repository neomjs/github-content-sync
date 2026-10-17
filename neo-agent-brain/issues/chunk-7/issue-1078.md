---
id: 1078
title: A Fleet Manager relaunch under running seats blocks new tool connections
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-10-17T16:11:53Z'
updatedAt: '2026-10-17T18:14:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1078'
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
blocking:
  - '[ ] 1081 Fleet quit and the continuity handoff share one budget'
closedAt: '2026-10-17T18:14:55Z'
---
# A Fleet Manager relaunch under running seats blocks new tool connections

## Context

Operator-observed 2026-10-17 00:5x Berlin (Tobi, the nightshift): after the Fleet Manager was relaunched at 22:40:06Z — its predecessor took SIGINT at 22:39:55Z (`main.log`) — every Claude seat's roster row read "New tool connections are blocked" (Sophie's read `skills not verified`, a different line). Running tools kept working; every new MCP connection of those seats — a fresh session, a bridge child's reconnect — was refused until the operator restarted each seat from the FM at 01:28–01:40 Berlin. Defect-note `f2650588` (Ada): 19 launchers pinned to `NEO_FLEET_LAUNCH_ISSUER` at the previous issuer's port, by then dead. Two facts from Ada's intake (`f42b7c0a`): (1) the installed bundle (Brain e219267e, staged 2026-10-14 21:26Z) predates PR #1043 (#1041, the bridge survives recoverable transport errors; merged 10-15) and PR #1064 (#1062, remote bridge sessions recover without replaying writes; merged 10-16) — that is why the bridges died at all, so two defects compounded that night; (2) Institution #424 names "the vessel is updated" as an ordinary failure the product must guide back to live — this ticket is its Brain half.

## The Problem

A seat launched by one Fleet process holds a launch-admission generation in that process (`ai/services/fleet/McpLaunchAdmissionService.mjs` `generations`, :112–115). A successor process restores it only after a proved handoff; otherwise `statusOf(agentId, {running: true})` answers `stale` / `ISSUER_REPLACED` (:752–760) and the Institution words it "New tool connections are blocked" (`apps/agentos/util/SeatLaunchAdmission.mjs:111`). Designed, documented and installed (#590 → #597, #655 → #660; the continuity path from Institution #674). On 2026-10-17 that path carried not one seat across an ordinary quit-and-relaunch, and nothing in `main.log` says which of its exits was taken. The operator's only remedy was nine seat restarts.

## The Architectural Reality

- The installed FM runs `ai/services/fleet/devFleetServer.mjs` (`FLEET_SERVER_ENTRY`, Institution `harness/brain.mjs:40`). Its `cleanShutdown` (:596–618): log `[fleet] received SIGINT; stopping.` → `server.drainFleetRequests(proofTimeoutMs)` → `lifecycle.handoffRuntimeCustody(...)` when `runtime.generation` is set → on a throw one generic line → `closeUnclean()` → `process.exit(0)`.
- The Institution tears the Brain child down with group SIGINT, a bounded grace of `graceMs = 10000` (`harness/brain.mjs:1150–1179`), then group SIGKILL. The drain alone may take `proofTimeoutMs = 10000` (`McpLaunchAdmissionService.mjs:100`), and `handoffRuntimeCustody` re-proves every surviving seat's owned credentials (`FleetLifecycleService.mjs:2316–2345`, `admission.prove` under the same timeout). The night's log: SIGINT at 22:39:55.455Z, the successor's plan line at 22:40:06.877Z — 11 s. A SIGKILL inside the handoff leaves the journal in `phase: 'active'`; the successor observes `handoff-not-completed` → `unclean-issuer-exit` (`launchAdmissionJournal.mjs:135–136`, `:338`) and keeps every generation `stale` with `RECOVERY_UNVERIFIED` (`recover`, :220–234).
- `recover()` has silent exits of its own: one unproved survivor is dropped in an empty catch (:259–262); a thrown recovery sets `continuityPhase = 'unavailable'` (:282) — today's 16:59 fleet start and the 01:28 restarts worked, so the phase was `'ready'`.
- Nothing logs any of it: `main.log` (4,975 lines) carries no `continuity`, `handoff`, `unclean-issuer-exit`, `handoff-replayed` or `recovery-unverified` word anywhere; the handoff's generic catch line is absent too. So either the handoff sealed and recovery failed silently, or the process died before the catch. The journal keeps the current state only (`issuer.json`, `fence.json`); the successor computes `previous.reason` at open and discards it.
- The launcher side is by design: every `.neo-fleet-mcp/<server>.sh` pins `NEO_FLEET_LAUNCH_ISSUER=http://127.0.0.1:<port>`; a successor re-binds the exact original origin only after a proved handoff (`listen({origin})`, :937–944) and never falls back to another port. A relaunch without a proved handoff therefore strands every pinned launcher until its seat restarts.

## The Fix

1. **The continuity dispositions reach the log and the roster.** The handoff logs its stage and reason (`drain`, `verify-seats` with the failing seat, `endpoint-binding`, `endpoint-release`, `sealed`); the successor logs `previous.reason` (`unclean-issuer-exit` / `handoff-replayed` / recovered) and, per retained generation, which check of :245–258 kept it stale. `statusOf` carries that reason instead of the bare `ISSUER_REPLACED`, so the Institution's existing "because <reason>" slot prints it.
2. ~~One budget for quit and handoff~~ → **#1081** (split 2026-10-17; the leaf `fleet.shutdownBudgetMs`, the host's serialized grace, the default from the measurement).
3. Once AC-3 names the exit a real relaunch takes, the mechanical fix for that exit lands under #1081 or a named successor — not before.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `devFleetServer.mjs` `cleanShutdown` (:596) | the quit path as it is | logs each handoff stage with its reason; the catch names the stage | the one generic line today | JSDoc | AC-1 |
| `McpLaunchAdmissionService#recover` (:207) | the recovery path as it is | logs `previous.reason` and per-generation stale reasons; keeps them on the generation's `reason` | `RECOVERY_UNVERIFIED` as today | class doc :43–75 | AC-2 |
| `statusOf()` (:752) → Institution `SeatLaunchAdmission.mjs:111` | the existing `{state, reason}` shape | `reason` stays `issuer-replaced` (a cross-repo vocabulary, `src/fleet/contract/launchAdmission.mjs`); a new `detail` beside it carries the specific exit (`unclean-issuer-exit`, `handoff-replayed`, the failing recovery check) — *amended at intake, PR #1080* | `detail: null` when no finer reason exists | CARD-CONTRACT.md control status (the Institution sibling prints `detail`) | AC-2, AC-3 |
| Institution `stopBrainChild` grace (`harness/brain.mjs:1155`) ↔ the handoff's budget | Institution #674's design | one budget: the handoff seals within the grace the host grants, or the host waits for the handoff's bound | SIGKILL after the budget, logged as `handoff-killed` with the elapsed time | `brain.mjs` §2.1.5 | AC-1, AC-4 |

Decision Record impact: aligned-with Institution #674 (an installed update keeps running seats on their generation); no ADR names the handoff budget.

> **Update 2026-10-17 17:0x Berlin (Clio, intake):** PR #1080 delivers Fix 1 (the dispositions reach the log and the status row as `detail`). **Update 19:3x Berlin (Euclid's acceptance-boundary read on #1080):** Fix 2 (one budget for quit and handoff) and its measurement are split out as **#1081** (blocked by this ticket) so that outcome has an owner and this ticket closes on Fix 1; former AC-4 lives there. AC-3 stays this ticket's post-merge validation, `[L4-deferred — operator handoff needed]`: the FM Quit → relaunch on the next candidate, read from the log and the roster; #1081 carries its witness.

## Acceptance Criteria

- [ ] AC-1 Unit (Brain): a quit whose handoff is interrupted at each stage (drain deadline, a failing seat proof, endpoint binding) logs that stage with its reason; a sealed handoff logs `sealed` with the origin it released.
- [ ] AC-2 Unit (Brain): a successor opening a journal in `phase: 'active'` logs `unclean-issuer-exit`, keeps the generations stale with that reason, and `statusOf(id, {running: true}).reason` carries it; a successor with a proved handoff and one unprovable survivor logs the failing check for that seat and restores the others.
- [ ] AC-3 Installed, post-merge: on the next Institution #12 candidate, an FM Quit → relaunch with nine running seats; every seat's roster line names the specific reason or none; when the handoff sealed, one fresh session on a Claude seat opens its MCP connections without a seat restart. Receipt on the Row 1 walk (Fri Oct 23).
- [ ] ~~AC-4 The handoff's measured duration with nine seats is recorded here and the grace/budget relation of Fix 2 is decided from it.~~ → moved to #1081 (2026-10-17); the measurement is AC-3's log line.

## Out of Scope

The bridge transport defects (#1041, #1062 — merged, awaiting the Monday candidate); the Stop-fleet dead end (#1074); the card wording itself (#590 / #655 shipped it); Start-fleet admission under load (#964); any fallback that lets a successor adopt a port it did not prove.

## Avoided Traps

Treating "New tool connections are blocked" as the bug (it is the designed, honest symptom of a lost handoff; the defects are the silence and the budget); relaxing the handoff's proofs to make it fast; a successor that re-issues admission without proof (the custody design forbids it); reading the night's report as the cause (the log window is the evidence; which exit fired stays unknown until AC-1/AC-2 speak).

## Related

Brain #1074 (the same night's other lifecycle dead end), Institution #424 (ordinary failures guided back to live — this is "the vessel is updated"), Institution #674 (the design: running seats keep their generation across an update), Institution #704 (an issuer lock outliving its process, the adjacent recovery doc), Institution #682 (restart every Claude seat for a staged update — the operator's workaround as a product step), Brain #964, Brain #1056 (the Fleet says why a seat exited without a Stop — the sibling "say why" precedent), defect-note `f2650588` (Ada).

Live latest-open sweep: checked the latest 20 open Brain issues at 2026-10-17 16:08:26Z (created-descending); nearest #1077 and #1074 (mine) — no equivalent. Exact sweeps: Brain `launch admission handoff / ISSUER_REPLACED / stale issuer / tool connections are blocked` → #980, #909, #1030 (closed admission work), #1056 (closed, the "say why" precedent); Institution `launch admission / tool connections / issuer` → #590, #655, #704, #674, #682 — none on the lost handoff at relaunch. A2A in-flight claims (all read-states, the last hour, at 16:09Z): none; Ada's `f42b7c0a` hands the filing to me. MC sweep on the symptom's nouns: 6 rows — my 2026-10-13 V-B-A of the class doc ("a new Fleet process holds no generation, so a seat it adopts reads stale until it is restarted"), the 10-10 Start-fleet trace; no prior decision on the quit budget or the silence. Own-assignment sweep: Brain #1077 #977 #850 #50 #51 #53, Institution #507 #505 #351 — none overlapping. Structure map: ran at Brain `dev` (exit 0); no new module — the owners are `ai/services/fleet/devFleetServer.mjs`, `McpLaunchAdmissionService.mjs`, `FleetLifecycleService.mjs` and Institution `harness/brain.mjs`.

unowned-rationale: filed on Ada's handoff from the design seat; the Fleet lifecycle steward self-selects (the continuity path is Sophie's #674 / #704 ground; the issuer trace is Ada's). The AC-3 witness on the Row 1 walk is mine.

Origin Session ID: 22306624-a5da-4d61-ac1f-2468c66f61e7

Retrieval Hint: "Fleet Manager relaunch running seats stale issuer handoff not completed New tool connections are blocked launch admission continuity"

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 22306624-a5da-4d61-ac1f-2468c66f61e7



## Timeline

- 2026-10-17T16:11:55Z @neo-fable-clio added the `bug` label
- 2026-10-17T16:11:55Z @neo-fable-clio added the `ai` label
- 2026-10-17T16:11:55Z @neo-fable-clio added the `agent-os` label
- 2026-10-17T16:58:06Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-10-17T17:06:18Z @neo-fable-clio cross-referenced by PR #1080
- 2026-10-17T17:25:41Z @neo-fable-clio cross-referenced by #1081
- 2026-10-17T17:25:56Z @neo-fable-clio marked this issue as blocking #1081
- 2026-10-17T17:31:17Z @neo-fable-clio referenced in commit `475d120` - "feat(fleet): name the recovery checks and witness each handoff stage (#1078)"
- 2026-10-17T17:35:47Z @neo-fable-clio referenced in commit `46ddc7b` - "test(fleet): a paused handoff phase keeps its stage as the last witness (#1078)"
- 2026-10-17T18:14:55Z @tobiu referenced in commit `6d47669` - "feat(fleet): the continuity path says what it did and why a seat stays stale (#1078) (#1080)

* feat(fleet): the continuity path says what it did and why a seat stays stale (#1078)

* feat(fleet): name the recovery checks and witness each handoff stage (#1078)

* test(fleet): a paused handoff phase keeps its stage as the last witness (#1078)"
- 2026-10-17T18:14:55Z @tobiu closed this issue
- 2026-10-17T18:17:43Z @neo-fable-clio cross-referenced by PR #1082

