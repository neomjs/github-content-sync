---
id: 660
title: A Fleet Manager quit kills every seat Fleet launched
state: CLOSED
labels:
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-01T09:12:51Z'
updatedAt: '2026-10-01T11:18:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/660'
author: neo-opus-grace
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-01T11:18:07Z'
---
# A Fleet Manager quit kills every seat Fleet launched

## Context

On 2026-09-30 the operator restarted the installed Fleet Manager to grant a macOS permission, and Sophie's Codex seat died with it (neomjs/neo-agent-institution#12, comment 5918679322). Moving existing peers into Fleet seats (#571) is frozen until a seat survives a quit (neomjs/neo-agent-institution#12, comment 5918767201). For a Claude seat the loss is worse than a restart: a kill between two tool calls drops that turn's memory save.

## The Problem

A seat is the Fleet server's child, and the Fleet server is the shell's child, so quitting the shell ends every seat:

- Institution `harness/appLifecycle.mjs:290`: will-quit → `teardownOwnedBrain` → `harness/brain.mjs:1036` `stopBrainChild`, which signals the Brain child's whole process group (SIGINT, then SIGKILL).
- Brain `ai/services/fleet/FleetLifecycleService.mjs:518`: seats spawn with `{stdio, env}` and no `detached`, so each one joins that group.
- `processes` (`:338`) is an in-memory Map. A restarted Fleet server knows nothing of a seat that did survive: it would show the seat stopped and offer a Start onto a profile that is already running.

## The Architectural Reality

- **Desktop seats can outlive the Fleet server; CLI seats cannot.** The class summary (`FleetLifecycleService.mjs:166-171`): the app-bundle families are stdin-indifferent and supervised by pid + SIGTERM; the CLI families exit on an EOF'd stdin, so a pipe held by the Fleet server is their liveness.
- **Re-adoption has Brain precedents.** `ProcessSupervisorService` adopts a PID file's process after `process.kill(pid, 0)` and an expected-command check (`ai/daemons/orchestrator/services/ProcessSupervisorService.mjs:207-244`). The wake daemon and the Codex Desktop helper scan pair a pid with its `ps -o lstart=` start time, so a reused pid is never mistaken for the original (`ai/daemons/wake/daemon.mjs:1512`, `ai/services/fleet/manageCodexDesktopRuntime.mjs:225`).
- **Codex Desktop's exact-profile proof stays.** Its Crashpad helpers already outlive the main process, and `finalizeCodexDesktopHelpers` (`:911`) proves zero residuals for one profile. An adopted Codex seat keeps that finalizer.
- **The shell's ownership rule already agrees.** The shell ADR binds the harness to stop "exactly the processes IT started" (Institution `harness/brain.mjs:7`, §2.1.1). The Fleet server started the seats, so once they leave its group the shell's teardown (§2.1.5) stops at the Fleet server with no shell change.

## The Fix

In `FleetLifecycleService`, one PR:

1. The app-bundle families (`claude-desktop`, `codex-desktop`, `antigravity`) spawn `detached`, with no stdio held by the Fleet server, and are `unref()`ed.
2. At spawn, a lease file in the seat's harness home records `agentId`, `pid`, its `lstart`, `harnessType`, the exact profile path and `startedAt`. It holds no secret and is removed when the seat stops or exits.
3. The first lifecycle read in a Fleet server process re-adopts each leased seat whose pid answers, was born at the leased `lstart`, and runs this seat's launch binary as `argv[0]` with the exact profile argument as a whole word: `running`, `adopted: true`, no child handle. A pid that is free or now belongs to a newer process (or a zombie) means the seat exited: the lease is removed and the agent reads `stopped` with why. An unreadable or foreign lease is invalid and removed. A pid that answers but cannot be identified is held: not running, not stopped, lease kept, Start refused, probed again on each read.
4. Stop on an adopted seat sends SIGTERM by pid, then SIGKILL after `sigkillTimeoutMs`, re-proving the seat's identity before each signal and on each poll. Codex Desktop still runs its helper finalizer, and an adopted Codex seat whose bundle no longer proves its Crashpad helper never reports a clean stop. A surviving seat whose birth time or lease cannot be persisted is stopped at once, and its Start fails with the reason.
5. Every other family spawns as today.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetLifecycleService#start` (app-bundle families) | `FleetLifecycleService.mjs:359` | spawns detached, unrefs, writes the lease | spawn-failure path unchanged | class summary, "Supervision transport" | unit arm on the injected spawn fn, red on dev |
| seat lease file | new, in `launch.instanceHome` | `{agentId, pid, pidStartedAt, harnessType, profile, startedAt}`, no secret | none written for other families | class summary | unit arm |
| re-adoption | new `FleetLifecycleService` method | verified leases become `running` + `adopted` | stale → removed and reported | JSDoc | unit arms: verified → adopted; reused pid → stale; other profile → stale |
| `FleetLifecycleService#status` | `:755` | adds `adopted` | `false` for spawned seats | JSDoc | unit arm |
| `FleetLifecycleService#stop` (adopted) | `:655` | SIGTERM by pid → SIGKILL; exit by polling | — | JSDoc | unit arm with injected kill/poll |

## Decision Record impact

aligned-with the Institution shell ADR §2.1.1 / §2.1.5 (one lifecycle owner, group teardown on quit): the shell's teardown is unchanged. Changes the Fleet supervision transport documented in the `FleetLifecycleService` class summary.

## Acceptance Criteria

- [ ] AC-1: App-bundle families spawn detached with no stdio held by the Fleet server and are unref'd; every other family spawns as today (unit arms on the injected spawn fn, red on dev).
- [ ] AC-2: A secret-free lease is written at spawn and removed at stop and at exit (unit).
- [ ] AC-3: A fresh service instance re-adopts only an exactly identified seat as `running` + `adopted`; a free or reused pid reads `stopped` with its lease removed; a longer or embedded profile, a helper or another program, and an uninspectable live pid are held unidentified with the lease kept and Start refused (unit, injected lookups).
- [ ] AC-4: Stop ends an adopted seat (SIGTERM, SIGKILL fallback, exit by polling) without ever signalling a pid a newer process took during the grace period; an adopted Codex Desktop seat still runs its helper finalizer, and without a provable helper reports cleanup unresolved until the proof returns (unit).
- [ ] AC-5: On a macOS checkout, an app-bundle seat started by a Fleet server keeps running after a group SIGKILL of that server; a new Fleet server adopts it and Stop ends it (L3 receipt).
- [ ] AC-6 `[L4-deferred — operator handoff needed]` (post-merge, installed): Sophie survives a quit and relaunch of the installed Fleet Manager, the roster shows her running (an observed row, not unmanaged), and Stop ends her — the exit proof on neomjs/neo-agent-institution#12; needs the Institution's Brain pin to move. Owned after merge by #571, receipt on neomjs/neo-agent-institution#12.

## Out of Scope

- CLI-family survival: it needs a stdin holder that outlives the Fleet server.
- The own-mode plane surviving a quit (neomjs/neo-agent-institution#12).
- A "Quit and stop all seats" menu item, and updating the README's quit note from neomjs/neo-agent-institution#369 (Institution, after AC-6).
- Re-arming a wake route for an adopted seat (#79).

## Avoided Traps

- `open -n -a` launches detach but leave no supervisable process; excluded by design (`deriveHarnessLaunchSpec.mjs:238`).
- Orphaning without a lease: the process survives but supervision does not — Emmy's condition on neomjs/neo-agent-institution#12.
- Trusting a bare pid: pids are reused; `lstart` pins it to one process.
- Teaching the shell to spare seats: the shell would have to know the Fleet server's children, which the one-lifecycle-owner rule rules out.

## Related

Parent #571 · neomjs/neo-agent-institution#12 (the lifecycle decision and exit proof) · neomjs/neo-agent-institution#369 (the current Quit note) · #79 (wake routes) · #584 (agents root).

Live latest-open sweep: latest 20 open Brain and Institution issues at 2026-10-01T09:11Z; no equivalent. A2A in-flight sweep: last 30 messages, all read states; no claim on this scope. Memory Core: the 2026-09-30 diagnosis and proposal on neomjs/neo-agent-institution#12; no prior decision against it. Own assignments: none on this surface. Structure map: `ai/services/fleet` owns it.

Origin Session ID: c4499e07-1e9b-4f4e-b876-d6afd7ea4364

Retrieval Hint: "Fleet Manager quit kills Fleet-launched seats" · "supervisor of record, not parent of record" · "seat lease re-adoption lstart"




## Timeline

- 2026-10-01T09:12:51Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-01T09:12:52Z @neo-opus-grace added the `enhancement` label
- 2026-10-01T09:12:52Z @neo-opus-grace added the `ai` label
- 2026-10-01T09:12:53Z @neo-opus-grace added the `architecture` label
- 2026-10-01T09:12:53Z @neo-opus-grace added the `agent-os` label
- 2026-10-01T09:12:58Z @neo-opus-grace added parent issue #571
- 2026-10-01T09:34:06Z @neo-opus-grace cross-referenced by PR #662
- 2026-10-01T09:34:28Z @neo-opus-grace cross-referenced by #571
- 2026-10-01T09:35:01Z @neo-fable-clio cross-referenced by #663
- 2026-10-01T09:40:25Z @neo-opus-grace referenced in commit `83acdcc` - "test(fleet): a lease named for another agent is never adopted, even when pid, start time and profile match (#660)"
- 2026-10-01T09:43:16Z @neo-opus-grace referenced in commit `6e474ef` - "feat(fleet): a seat that exited unsupervised reads as an observed stop in the fleet view, with its reason (#660)"
- 2026-10-01T10:43:51Z @neo-opus-grace referenced in commit `f179bef` - "fix(fleet): an adopted seat is signalled only while its identity is proven, and a seat that cannot be leased does not run (#660)

Emmy's Round-1 review (5378031191) reproduced four failure paths:

- ownership is now exact: the launch binary as argv[0] (a script behind its interpreter)
  and the profile argument as a whole word, re-proven before every signal and on every
  poll, so a pid taken by a newer process never receives the seat's SIGKILL;
- a pid that answers but cannot be identified is held: not running, not stopped, lease
  kept, Start refused, re-probed on each read; a zombie counts as exited;
- a surviving seat whose birth time or lease cannot be persisted is stopped while this
  server still holds it, and its Start fails with the reason;
- an adopted Codex Desktop seat without a provable Crashpad helper never reports a
  clean stop, and Stop retries the proof.

An unreadable or foreign lease is invalid rather than an exit."
- 2026-10-01T11:02:02Z @neo-opus-grace referenced in commit `5135304` - "fix(fleet): a launched script counts as the seat only behind the interpreter its own shebang names (#660)"
- 2026-10-01T11:18:07Z @tobiu referenced in commit `6eaa779` - "feat(fleet): an app-bundle seat outlives the Fleet server and is re-adopted from its lease (#660) (#662)

* feat(fleet): an app-bundle seat outlives the Fleet server and is re-adopted from its lease (#660)

Quitting the Fleet Manager tore down the Fleet server's process group, and every seat
Fleet had spawned sat in that group. The app-bundle families (claude-desktop,
codex-desktop, antigravity) are stdin-indifferent, so they now spawn detached with no
stdio held by the server and leave a lease in their harness home: pid, its ps start
time, startedAt and the checkout. A later Fleet server re-adopts a seat only when the
live process still matches: the pid answers, the start time is the leased one, and the
command line carries the profile the agent's home derives. Commands, profile, homes
and the Codex Crashpad helper are re-derived from the registry and AiConfig, never
read from the lease, which the seat itself can write. Stop on an adopted seat signals
by pid and polls for the exit; CLI families keep the held stdin they need to live.

* test(fleet): a lease named for another agent is never adopted, even when pid, start time and profile match (#660)

* feat(fleet): a seat that exited unsupervised reads as an observed stop in the fleet view, with its reason (#660)

* fix(fleet): an adopted seat is signalled only while its identity is proven, and a seat that cannot be leased does not run (#660)

Emmy's Round-1 review (5378031191) reproduced four failure paths:

- ownership is now exact: the launch binary as argv[0] (a script behind its interpreter)
  and the profile argument as a whole word, re-proven before every signal and on every
  poll, so a pid taken by a newer process never receives the seat's SIGKILL;
- a pid that answers but cannot be identified is held: not running, not stopped, lease
  kept, Start refused, re-probed on each read; a zombie counts as exited;
- a surviving seat whose birth time or lease cannot be persisted is stopped while this
  server still holds it, and its Start fails with the reason;
- an adopted Codex Desktop seat without a provable Crashpad helper never reports a
  clean stop, and Stop retries the proof.

An unreadable or foreign lease is invalid rather than an exit.

* fix(fleet): a launched script counts as the seat only behind the interpreter its own shebang names (#660)"
- 2026-10-01T11:18:08Z @tobiu closed this issue

