---
id: 942
title: The operator can Skip a pending Start's dependency install
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-08T22:44:30Z'
updatedAt: '2026-10-09T12:04:20Z'
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
closedAt: '2026-10-09T12:04:20Z'
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
| `skipAgentDependencies` (new; `src/fleet/contract/wire.mjs`, `fleetServerPolicy.mjs`, `FleetControlBridge`) | neomjs/neo-agent-institution#610 design read; this ticket | A lifecycle-write verb taking the seat id, answering `{id, skippedStarts}` | An older Fleet refuses the unknown method, failing closed | JSDoc | wire/policy parity specs, bridge unit |
| `FleetLifecycleService.skipDependencies(id)` (new) | same | Aborts the skip controller of each pending Start of `id` not yet skipped; `skippedStarts` counts them. A Start already past its install ignores it | No pending Start: `skippedStarts: 0`, no change. A repeated Skip does not count again | JSDoc | lifecycle unit |
| `FleetManager.startAgent` → `startAgentProvisioned({dependencySkipSignal})` | #938's seam | Each attempt's own skip signal reaches `installSeatDependencies`: the interrupted checkouts read `skipped`, finished ones keep their outcome, and the Start launches | A Stop still wins: its interrupted rows read `canceled` and the Start answers `canceledStart`. A later Start gets a fresh skip signal | JSDoc | manager unit with an injected install |

Decision Record impact: none (completes #937's accepted contract).

## Acceptance Criteria

- [ ] `skipAgentDependencies` is a Fleet wire method with `stopAgent`'s policy rows, reaching `FleetLifecycleService` through the bridge and the manager.
- [ ] A Skip while a Start installs ends the running installs: the interrupted checkouts read `skipped`, finished ones keep their outcome, and the Start launches (unit over `FleetManager.startAgent` with an injected install).
- [ ] A Skip with no pending Start answers `skippedStarts: 0` and changes nothing; a Start after a Skip gets a fresh skip signal.
- [ ] A Stop after a Skip still cancels the Start: it answers `canceledStart`, and the rows it interrupted read `canceled`.
- [ ] Post-merge (installed): the Agent Detail Skip (neomjs/neo-agent-institution#616) ends a fresh seat's install and the seat launches. Residual owner: neomjs/neo-agent-institution#616.

## Out of Scope

- The Agent Detail control and the progress line (neomjs/neo-agent-institution#616), and the readiness exception (neomjs/neo-agent-institution#610).
- A Skip chosen before the Start: the design read offers Skip during the install only.
- Stop's semantics.

## Avoided Traps

- **Skip as a Start option** (`startAgent(id, {skipDependencies})`), neomjs/neo-agent-institution#610's first shape: the design read moved Skip into the install, where the operator can see it is slow. A Start option asks before any cost is visible.
- **Reusing the Stop signal:** Stop cancels the Start, and Skip must let it launch (Grace's ask on #937).

## Related

#937 / PR #938 (producer) · neomjs/neo-agent-institution#616 (consumer, Grace; split from neomjs/neo-agent-institution#610) · neomjs/neo-agent-institution#245 (operator decision) · #571

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
- 2026-10-09T03:33:30Z @neo-opus-grace cross-referenced by #616
- 2026-10-09T03:56:52Z @neo-opus-vega cross-referenced by #610
- 2026-10-09T05:14:34Z @neo-opus-grace cross-referenced by PR #629
- 2026-10-09T06:17:19Z @neo-opus-grace cross-referenced by #633
- 2026-10-09T12:04:21Z @tobiu referenced in commit `e74cd6f` - "feat(fleet): the operator can Skip a pending Start's dependency install (#942) (#943)

* feat(fleet): the operator can Skip a pending Start's dependency install (#942)

#938 gave `installSeatDependencies` a skip signal separate from Stop, but nothing fired it.

- `FleetLifecycleService`: each pending Start owns a Stop fence and a Skip fence. `skipDependencies(id)`
  aborts the Skip fence of every pending Start of the seat and answers `{id, skippedStarts}`;
  `dependencySkipSignal(id, signal)` hands one attempt's fence out.
- `FleetManager.startAgent` passes it to the provisioning composer as `dependencySkipSignal`, and
  `skipAgentDependencies(id)` delegates to the lifecycle like `stopAgent`.
- `skipAgentDependencies` joins `FLEET_WIRE_METHODS`, both policy ledgers (`awaiting-s5`,
  `lifecycle-write`, as `stopAgent`) and `FleetControlBridge`.
- `createFleetRegistryBridge`'s note no longer names a wire-method twin and a parity lint: the
  product bridge imports this same list.

* test(fleet): the Skip test dispatches through the real wire and bridge (#942)"
- 2026-10-09T12:04:21Z @tobiu closed this issue
- 2026-10-09T12:52:30Z @tobiu referenced in commit `7ae86bd` - "feat(agentos): a Start the Fleet is preparing shows its count on the card, cancels from the toggle, and skips from Repository (#616) (#629)

* feat(agentos): a running seat whose working checkout was not prepared says so, and each checkout's preparation reads in Repository (#610)

SeatDependencies words the Fleet's dependencyOutcomes once for the card,
Agent Detail's Repository pane and the Accounts Repositories card: the card's
quietest exception reads "skills not verified" for a running seat whose working
checkout the last start did not prepare; every checkout keeps its own row, a
failed clone first, an install still running as this start.

* test(agentos): the preparation frames, and the card's skills line follows the state it shows (#610)

The skills line gates on the resolved display state, so a seat the card reads
offline (stopped or unobserved) says nothing. New goldens: the card line and
Detail's Repository pane settled, live and light; Ada's Repository pane now
carries her clone outcomes in two refreshed goldens. Baselines re-stamped.

* test(agentos): re-stamp the visual baselines over dev's merged inputs (#610)

* feat(agentos): a Start the Fleet is preparing shows its count on the card, cancels from the toggle, and skips from Repository (#616)

While the Fleet reports installs for a pending Start, the card reads
`start… preparing dependencies (n/m done)` in place of `start…` and the local
`no answer yet`, and the power verb is Cancel start; a sent cancel reads
`canceling start…`. The Repository pane offers Skip while the Start installs,
with its consequence beside it, as a side request that never claims the Start's
pending verb. The verb comes from the pinned wire, so Skip appears once a pin
carries neomjs/neo-agent-brain#942. The pane's body becomes its own component,
which keeps detail/Container.mjs under the app file-size bar.

* test(agentos): the preparing card and the Repository pane's Skip, both skins (#616)

* test(agentos): the Skip driver's summary names the verb it adds, not the pin that ships it (#616)

* test(agentos): the card falls back to its plain pending and timeout text once the live rows go (#616)

* fix(agentos): a checkout's dependency row speaks over an older clone outcome, and a clone with no install reported reads unverified (#610)

* fix(agentos): a clone with no install reported reads unverified in one line, and the goldens re-capture the panes that showed it (#610)

* refactor(agentos): the panes read a live Start through the same line the card counts (#616)

* refactor(agentos): Detail's Repository pane reads its checkouts through the Accounts card's Store/Model/list, read-only (#610)

* fix(agentos): the Repository body takes its content's height, the list keeps the pane's rhythm and the failure's weight (#610)

* docs(agentos): the fold's reason names a checkout the start held, not one it cloned (#610)

* fix(agentos): a Skip binds to the attempt it was asked in, and an unanswered cancel stays canceling (#616)

* test(agentos): the earlier attempt's late Skip answer never replaces the current request (#616)

* test(agentos): re-stamp the visual baselines over the attempt-bound Skip and the open cancel (#616)

* test(agentos): the drill round-trip's narrow Detail frames show Ada's checkout rows (#610)"

