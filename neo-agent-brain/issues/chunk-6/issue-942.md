---
id: 942
title: The operator can Skip a pending Start's dependency install
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-08T22:44:30Z'
updatedAt: '2026-10-08T23:19:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/942'
author: neo-opus-vega
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
# The operator can Skip a pending Start's dependency install

## Context

Brain #937 (PR #938, merged 2026-10-08) makes a Fleet Start install each seat checkout's locked dependencies before launch. Its installer already honors a Skip: `installSeatDependencies` takes a `skipSignal` separate from the Start's Stop signal, ends the running `npm` installs, waits for them to exit, marks the interrupted checkouts `skipped` and lets the launch go on (`ai/services/fleet/installAgentRepoDependencies.mjs`). `startAgentProvisioned` accepts it as `dependencySkipSignal`. Nothing produces that signal: `FleetManager.startAgent` passes none, and no Fleet verb reaches it.

The operator's 2026-09-30 decision asked for preparation "with visible progress and a skip option" (Emmy's record on neomjs/neo-agent-institution#245). Grace's design read on neomjs/neo-agent-institution#610 ([comment](https://github.com/neomjs/neo-agent-institution/issues/610#issuecomment-6067909778)) places Skip on Agent Detail's Repository pane, offered during the install: Skip continues the launch, Stop keeps `canceledStart`. Grace asked for the producer seam in #937 and for the verb as this Brain leaf.

## The Problem

**How this started (operator, 2026-10-08):** on a first-ever agent boot the harness requires a new login. Vega's first Start outlived the cockpit's 30 s rule while that happened, and the roster card kept `start… stale — no response` beside a working seat (neomjs/neo-agent-institution#608; neomjs/neo-agent-institution#609 now says `no answer yet` and reconciles the late answer). #937 then added up to three `npm ci` runs to the same first Start. Skip lets the operator end that added wait and launch now. Without it, Stop is the only way out, and Stop cancels the Start; #610's Skip control has no verb to call.

## The Architectural Reality

- Skip mirrors the Stop path: `FLEET_WIRE_METHODS` (`src/fleet/contract/wire.mjs`) → `fleetServerPolicy.mjs` (`stopAgent`: `awaiting-s5`, `lifecycle-write`) → `FleetControlBridge.stopAgent` → `FleetManager.stopAgent` → `FleetLifecycleService.stop`, which aborts the controller of every pending Start in `pendingStarts`.
- `FleetLifecycleService.beginStart(id)` registers a pending Start (one `AbortController` per attempt, keyed by its signal) and `finishStart(id, signal)` releases it. `FleetManager.startAgent` passes that signal to `startAgentProvisioned` as `startSignal`.
- `startAgentProvisioned` hands `dependencySkipSignal` to `installSeatDependencies` as `skipSignal` and records live and final rows through `setPendingDependencies`.

## The Fix

1. `FleetLifecycleService`: each pending Start also owns a dependency-skip controller, created in `beginStart` and released in `finishStart`. `skipDependencies(id)` aborts it for every pending Start of the seat that has not been skipped yet and answers how many it reached. A seat with no pending Start answers zero and nothing changes.
2. `FleetManager.startAgent` passes the attempt's skip signal to the provisioning composer as `dependencySkipSignal`. `FleetManager.skipAgentDependencies(id)` delegates to the lifecycle, like `stopAgent`.
3. The verb: `skipAgentDependencies` in `FLEET_WIRE_METHODS`, its `fleetServerPolicy` rows (`awaiting-s5`, `lifecycle-write`, as `stopAgent`), and `FleetControlBridge.skipAgentDependencies(id)`.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `skipAgentDependencies` (new; `src/fleet/contract/wire.mjs`, `fleetServerPolicy.mjs`, `FleetControlBridge`) | #610 design read; this ticket | A lifecycle-write verb taking the seat id, answering `{id, skippedStarts}` | An older Fleet refuses the unknown method, failing closed | JSDoc | wire/policy parity specs, bridge unit |
| `FleetLifecycleService.skipDependencies(id)` (new) | same | Aborts the skip controller of each pending Start of `id` not yet skipped; `skippedStarts` counts them. A Start already past its install ignores it | No pending Start: `skippedStarts: 0`, no change. A repeated Skip does not count again | JSDoc | lifecycle unit |
| `FleetManager.startAgent` → `startAgentProvisioned({dependencySkipSignal})` | #938's seam | Each attempt's own skip signal reaches `installSeatDependencies`: the interrupted checkouts read `skipped`, finished ones keep their outcome, and the Start launches | A Stop still wins: its interrupted rows read `canceled` and the Start answers `canceledStart`. A later Start gets a fresh skip signal | JSDoc | manager unit with an injected install |

Decision Record impact: none (completes #937's accepted contract).

## Acceptance Criteria

- [ ] `skipAgentDependencies` is a Fleet wire method with `stopAgent`'s policy rows, reaching `FleetLifecycleService` through the bridge and the manager.
- [ ] A Skip while a Start installs ends the running installs: the interrupted checkouts read `skipped`, finished ones keep their outcome, and the Start launches (unit over `FleetManager.startAgent` with an injected install).
- [ ] A Skip with no pending Start answers `skippedStarts: 0` and changes nothing; a Start after a Skip gets a fresh skip signal.
- [ ] A Stop after a Skip still cancels the Start: it answers `canceledStart`, and the rows it interrupted read `canceled`.
- [ ] Post-merge (installed): #610's Skip on Agent Detail ends a fresh seat's install and the seat launches. Residual owner: neomjs/neo-agent-institution#610.

## Out of Scope

- The Agent Detail control, the progress line and the readiness exception (neomjs/neo-agent-institution#610).
- A Skip chosen before the Start: the design read offers Skip during the install only.
- Stop's semantics.

## Avoided Traps

- **Skip as a Start option** (`startAgent(id, {skipDependencies})`), #610's first shape: the design read moved Skip into the install, where the operator can see it is slow. A Start option asks before any cost is visible.
- **Reusing the Stop signal:** Stop cancels the Start, and Skip must let it launch (Grace's ask on #937).

## Related

#937 / PR #938 (producer) · neomjs/neo-agent-institution#610 (consumer, Grace) · neomjs/neo-agent-institution#245 (operator decision) · #571

Live latest-open sweep: the latest 20 open Brain issues at 22:43Z; no equivalent. Org-wide `gh search issues "skip dependency install"`: none.
A2A claim sweep: no claim on a Skip verb in the last hour; Grace's 19:55Z message names this leaf as Brain's.
MC sweep: "operator cannot skip the npm ci dependency install while a Fleet Start waits", 6 results: the 2026-09-30 decision (default preparation with progress and a skip option) and the #937 session; no contrary decision.
Own-assignment sweep: 14 open, none overlapping.
Structure map: no new file; `ai/services/fleet` and `src/fleet/contract` own every touched surface.

Origin Session ID: 7d3fc6b2-cee6-4f82-ba2c-103729d4047a
Retrieval Hint: "skipAgentDependencies dependencySkipSignal Skip pending Start install Agent Detail #610"



## Timeline

- 2026-10-08T22:44:31Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-08T22:44:32Z @neo-opus-vega added the `enhancement` label
- 2026-10-08T22:44:32Z @neo-opus-vega added the `ai` label
- 2026-10-08T22:44:33Z @neo-opus-vega added the `agent-os` label
- 2026-10-08T22:57:02Z @neo-opus-vega referenced in commit `99e5c0b` - "test(fleet): the Skip test dispatches through the real wire and bridge (#942)"
- 2026-10-08T22:57:22Z @neo-opus-vega cross-referenced by PR #943

