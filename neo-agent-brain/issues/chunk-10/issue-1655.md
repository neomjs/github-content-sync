---
id: 1655
title: Serial survivor validation lets one slow seat fail the whole handoff
state: OPEN
labels:
  - bug
  - ai
  - performance
assignees:
  - neo-opus-ada
createdAt: '2026-11-12T17:50:16Z'
updatedAt: '2026-11-12T18:09:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1655'
author: neo-opus-ada
commentsCount: 1
parentIssue: 732
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
# Serial survivor validation lets one slow seat fail the whole handoff

## Context

The FM install at 2026-11-12T17:46Z (Brain `f25fb9b9` → `dc3082cc`) ended with every running seat `stale`. `main.log`:

```
17:46:18.341Z survivor neo-gpt-sophie: verified (77 ms / 8000 ms budget)
17:46:19.128Z survivor neo-opus-ada: verified (864 ms / 8000 ms budget)
17:46:25.954Z survivor neo-opus-grace: verified (7690 ms / 8000 ms budget)
17:46:26.266Z refused at survivor-validation: deadline (8002 ms / 8000 ms budget)
17:47:30.858Z continuity recovery: previous issuer unclean-issuer-exit … neo-opus-ada stale, neo-opus-grace stale, neo-opus-vega stale, …
```

One seat took 6.8 s, and the seats after it never got a turn. The same day's earlier installs, all with 6–8 running seats:

| Install | Seats | Survivor validation | Slowest seat |
|---|---|---|---|
| 02:00Z | 8 | sealed 5855 ms | 1.9 s |
| 13:55Z | 7 | sealed 3425 ms | 1.3 s |
| 16:43Z | 6 | sealed **7908 ms** | 5.6 s (vega) |
| 17:22Z | 6 | sealed 2878 ms | 1.0 s |
| 17:46Z | 6 | **refused at 8002 ms** | ≥ 6.8 s (grace) |

Claude and Gemini seats cost 0.6–1.9 s each when nothing is slow, and GPT seats 10–80 ms. On 2026-11-11 at 17:52Z, with the plane down, the same validation took 149 ms for 8 seats (about 5 ms each). So the cost sits in the proofs against live credential owners. That is an inference from these two readings; no per-proof timing exists yet. The 17:46Z install ran 2 min after a plane cut (17:44:26Z), which may have made those proofs slower.

## The Problem

The issuer refuses the whole handoff when one deadline expires (correct, per the ADR below). But it spends that deadline in a sum: seat after seat, each awaiting its own proofs. Two seats at 4 s, or one at 7 s behind two ordinary ones, run out the budget. At about nine Claude/Gemini seats, the ordinary cost alone reaches it. A refused handoff strands every running seat: their MCPs keep answering, but launch admission ends, and each needs a Stop + Start to rebind.

**Design authority:** `FleetLifecycleService#handoffRuntimeCustody`'s JSDoc, "one seat never blocks the update", and Institution ADR 0034 §2.5 *Runtime generations for managed seats* item 4: "An unproved fact about one surviving seat revokes only the authority that seat would have carried." On 2026-11-12, one slow seat blocked the update for every seat.

## The Architectural Reality

- `FleetLifecycleService#handoffRuntimeCustody` builds `verifySeats`: a `for … of this.processes.values()` loop that `await`s `verifySurvivor` per record, then applies that seat's revocations and witness line.
- `McpLaunchAdmissionService#performHandoff` races `verifySeats` against one `deadline` timer (the whole-quit budget), then proof-drain, endpoint-release and durable-handoff under the same race.
- The budget is `NEO_FLEET_SHUTDOWN_BUDGET_MS = 8000`, which is the host grace of 10000 minus a 2000 margin (Institution `harness/brain.mjs`, `BRAIN_SHUTDOWN_GRACE_MS` / `BRAIN_SHUTDOWN_MARGIN_MS`). Each proof is bounded by `proofTimeoutMs = 10000`, longer than the whole budget.
- Shared work: `verifySurvivor`'s `verified` map holds one runtime verify per distinct pin across seats. `prove` dedupes only within one seat's admission generation (`{owners, proofs} = generation`), so each seat's proofs are its own.

## The Fix

Run each survivor's `verifySurvivor` concurrently and apply each seat's outcome as it settles: revocations, `runtimeUnresolved`, and its witness line. The survivor stage then costs about the slowest seat instead of the sum. The 17:46Z install would have needed about 6.8 s, and 16:43Z about 5.6 s. The deadline, its value and its issuer-level refusal stay as they are. A `pendingStarts` refusal and the no-runtime-Start throw stay first, before any proof starts.

## Acceptance Criteria

- [ ] Three surviving seats whose verify takes 3 s each seal under an 8 s deadline. Red on `dc3082cc`: refused at survivor-validation at 8 s. Unit, `FleetLifecycleService` handoff spec.
- [ ] Per-seat semantics are unchanged: an unprovable seat or server revokes only its own authority and keeps its witness line, and a failed durable revocation still refuses the handoff. The existing handoff specs pass unchanged, except where they pinned the serial order of proofs or witness lines.
- [ ] Concurrent seats on one runtime pin run one runtime verify. Unit.
- [ ] *(post-merge, installed)* The next FM install with ≥ 6 running seats seals. Its receipt records the survivor-validation time beside the slowest seat's.

## Out of Scope

- Degrading a seat at the deadline instead of refusing. ADR 0034 item 4 names "the deadline" as an issuer refusal, so that is an ADR amendment. #1185 deferred it until per-seat timings existed; the table above is that evidence, if concurrency proves not enough.
- The per-proof bound and the budget value (#1081).
- The order of a plane cut and an FM install in our own procedure.
- Rebinding the seats left stale by 17:46Z (each owner's Stop + Start).

## Avoided Traps

- **Raising the budget.** The host kills the Fleet child after its 10 s grace, so a longer handoff is a SIGKILL, not a seal.
- **A worker pool or limit.** Running seats stay ≤ 12 today, and a pool adds a knob that nothing reads.

## Related

#1508 (FM MVP epic) · #1185, #1143, #1552 (earlier whole-handoff refusals, each fixed per cause) · #1081 (the shared budget) · defect-note `26a8b351` (the 16:43Z near miss).

Decision Record impact: aligned-with Institution ADR 0034 §2.5 runtime-generations item 4 and ADR 0038 §2.5.2. The refusal set is unchanged.

Sweeps, 2026-11-12T17:49Z: latest 20 open (157 open, red band), with no equivalent. Closed searches `continuity handoff deadline`, `survivor validation` and `handoff concurrent seats` returned #1081, #1143, #1185 and #1552; I read their Out of Scope and Avoided Traps, and none covers serial validation. A2A: the 30 newest messages, all read-states, with no claim on this surface. MC: one query, no prior decision. Own-assignment: this lands under my #1508. Structure map: `npm run ai:structure-map -- --files --loc` exit 0. Both files exist in `ai/services/fleet/`, and no module is added.

Not a good first issue: it changes the FM update path every seat depends on.

Origin Session ID: 6c7d5652-1e67-41ec-a9af-2e828bfe463e
Retrieval Hint: "continuity handoff refused at survivor-validation deadline, one slow seat, serial verifySurvivor, seats stale"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-11-12T17:50:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T17:50:18Z @neo-opus-ada added the `bug` label
- 2026-11-12T17:50:18Z @neo-opus-ada added the `ai` label
- 2026-11-12T17:50:18Z @neo-opus-ada added the `performance` label
- 2026-11-12T17:50:53Z @neo-opus-ada added parent issue #732
- 2026-11-12T17:59:32Z @neo-opus-ada cross-referenced by PR #1656
### @neo-opus-ada - 2026-11-12T18:09:05Z

**Handover (session refresh; the owner stays me):** PR #1656 is at `cca4bdb7` with every check green and `CLEAN`, and Euclid (GPT) is requested as the cross-family reviewer.

**Pickup:** on Euclid's review, address any Required Actions on the same branch. On approval, hand the merge to @tobiu. After the merge, an FM build that carries #1656, then AC-4: an install with ≥ 6 running seats seals, with the survivor-validation time beside the slowest seat's. That same install rebinds nothing by itself; the seats left stale at 17:46Z rebind at their own Stop + Start.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


