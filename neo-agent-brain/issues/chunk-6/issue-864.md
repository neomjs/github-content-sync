---
id: 864
title: 'Configuration offers the models and efforts a seat''s harness names, and Start refuses one it lacks'
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-04T20:07:35Z'
updatedAt: '2026-10-05T10:21:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/864'
author: neo-opus-vega
commentsCount: 0
parentIssue: 867
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-05T10:21:56Z'
---
# Configuration offers the models and efforts a seat's harness names, and Start refuses one it lacks

Sub of #571 · split from #862, whose record and Start writes stay there · design read: Clio, [#571 comment 5983426039](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-5983426039) · Codex contract: Emmy, [#862 comment 5983659841](https://github.com/neomjs/neo-agent-brain/issues/862#issuecomment-5983659841) · consumer: Institution #559

## Context

#862 lets an operator declare the model and reasoning effort a seat's harness starts on. Clio's design read says the values offered are the harness's own catalog, and the product never invents a model list. Start also needs one refusal: a declared model the harness does not offer must refuse in the design read's words (`start refused: model <x> is not available — change it in Detail › Configuration`), and the roster card shows it. Both need a read of what the harness names. Nothing in Fleet reads that today.

## The Architectural Reality

- **Codex** (both families). The app-server's paged `model/list` (`cursor`, `limit`, `includeHidden`) gives each model `supportedReasoningEfforts`, `defaultReasoningEffort`, `hidden` and `isDefault`. An effort is an open string whose valid values depend on the model. Its answer depends on the client and the account, so it is read in the seat's own harness context (its `CODEX_HOME`). Source: Emmy's read of codex-cli 0.160.0's generated schema, linked above.
- **The protocol, run 2026-10-04 against codex-cli 0.160.0.** On stdio, one JSON message per line:
  1. `initialize` with `{clientInfo: {name, title, version}, capabilities}`;
  2. the `initialized` notification;
  3. `model/list` with `includeHidden: true`, followed through `nextCursor`.

  Run against an empty temporary home with no login, it answered a bundled catalog of 11 models. Three were hidden, and efforts differed per model (`gpt-6-luna` stops at `max`, `gpt-5.5` at `xhigh`). So a catalog read is not an entitlement receipt, as Emmy's read says.
- **A read writes.** The app-server creates SQLite state (`logs_2.sqlite`, `goals_1.sqlite`, `memories_1.sqlite`, `installation_id`) in the `CODEX_HOME` it is given. A read in a seat's own home therefore runs before the seat's harness starts and holds that home until its process has exited. It never starts a second server beside a running one: a running seat answers the catalog its last start read.
- **`claude-code`.** Its CLI names its effort levels in `--help` (`low`, `medium`, `high`, `xhigh`, `max`, CLI 2.1.212) and accepts a model alias or full id. No list verb is known.
- **`claude-desktop`.** It takes no declaration (#862 AC-3), so it has nothing to offer.
- A Fleet-side list would drift, and ADR-0019 §3 rules out hidden defaults.

## The Fix

1. A Fleet read, per seat, that answers `{state, models: [{id, efforts, defaultEffort, hidden}]}` from the harness:
   - Codex families: `model/list` with `includeHidden`, every page.
   - `claude-code`: its named levels, with the model left free.
2. Every provisioned start reads it, declared or not, and keeps it, so a seat started on its defaults can be changed in Configuration while it runs.
3. Start refuses a declared model, or an effort that model lacks, only when the read is complete and the value is missing from it. A failed, incomplete or unreadable read is unknown and refuses nothing. Catalog membership is not an entitlement receipt either.
4. A read is over only when its app-server has exited, and reads and starts on one seat take its home one at a time.

### Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| Wire verb `fleetSeatModelCatalog({id})` | `FleetManager.fleetSeatModelCatalog`, `FleetControlBridge`, `wire.mjs`, `fleetServerPolicy` | A stopped seat is read fresh while it holds the seat's home; a running seat answers the catalog its last start kept, with `observedAt`. Classed `lifecycle-write` / S4 with the owner principal, because a read runs a write-capable app-server | An unknown seat, a failed read, or a seat running since before this Fleet started answers `unavailable` with its reason; never a refusal | the method's JSDoc | `FleetManager.spec`, `FleetControlBridge.spec`, `dispatchFleetRequest.spec`, `fleetServer.spec` |
| Catalog answer `{state, models: [{id, slug, efforts, defaultEffort, hidden, isDefault}], efforts?, reason, observedAt?}` | `seatModelCatalog.readSeatModelCatalog` (family dispatch), `codexModelCatalog.readCodexModelCatalog` | `complete` only when every page was read and readable; `claude-code` answers `efforts` and leaves the model free; other families answer `unsupported` | `partial` keeps the pages before a failure or an unreadable page or row; `unavailable` when nothing was read, or the Codex home has no login | the modules' JSDoc | `codexModelCatalog.spec`, `seatModelCatalog.spec` |
| Read lifetime and the seat's home | `readCodexModelCatalog`, `FleetManager.withSeatHome` | The answer is handed over after the app-server exits: SIGTERM, then SIGKILL after a bounded grace. Reads and starts on one seat run one at a time, the running check inside the hold | An app-server that outlives both signals answers `stillRunning`, and Start refuses `FLEET_SEAT_HOME_IN_USE`, nothing cloned or launched | the functions' JSDoc | `codexModelCatalog.spec`, `FleetManager.spec`, `startAgentProvisioned.spec` |
| Start's read and refusal | `startAgentProvisioned` | Read at every provisioned start (repository or not) after the identity check, kept for Configuration; a declared value a complete catalog lacks refuses `FLEET_SEAT_MODEL_UNAVAILABLE`: nothing cloned or configured, the harness not started. The Codex read itself runs the app-server in the seat's home, which writes its own state there | Any read that is not complete refuses nothing | the function's JSDoc | `startAgentProvisioned.spec` |
| Status `seatModel` `{state, model, reasoningEffort, reason}` | `FleetLifecycleService.setSeatModel` / `seatModelOf` → `status` → runtime row → cockpit | What the seat's latest start found of its declaration: `refused` with the reason, or the read's state. Cleared at the start of every provisioned start, so it never outlives the start it describes | `null` when nothing was declared or nothing was read | the methods' JSDoc | `FleetLifecycleService.spec`, `startAgentProvisioned.spec`, `FleetManager.spec` |
| Shared harness metadata `seatSettings`, `resolveHarnessSeatSettings(type)` | `src/fleet/contract/harnessTypes.mjs` | `'codex-config'` (`codex`, `codex-desktop`), `'args'` (`claude-code`), `null` otherwise: the one fact the declaration normalizer, the catalog read and the Institution's Seat group share | An unknown type resolves to `null`: no declaration | the catalog's JSDoc | `fleetContract.spec` |
| Consumer | Institution #559 | Configuration's Seat group offers the catalog; the roster card reads `seatModel` when a start was refused | — | #559 body | #559's ACs |
| Acceptance evidence | this ticket's ACs | Unit and stub evidence only | Installed acceptance rides Institution #559 AC-4 and #862 AC-5 on the next #12 candidate; a catalog is never an entitlement receipt | this ticket | post-merge |

## Acceptance Criteria

- [ ] AC-1: the read answers each Codex seat's catalog through its own harness context, hidden entries included and flagged, every page followed. A failed, partial or unreadable read answers its state, never an empty or complete catalog. Unit, with a stub app-server.
- [ ] AC-2: Start refuses a declared model, or an effort the declared model lacks, with the design read's words, and only on a complete read. Every start keeps the catalog it read, so a seat started on its defaults offers it while it runs. Unit.
- [ ] AC-3: no list is Fleet's own: the offered values are what the harness answered, and `claude-code`'s levels are the ones its CLI names. Unit.
- [ ] AC-4: a read is over only when its app-server has exited, and reads and starts on one seat never overlap. An app-server that outlives both signals refuses the start. Unit, with delayed-exit, signal-ignoring and unkillable children, and concurrent reads and a start.

## Out of Scope

- Declaring, writing and reading back the values: #862.
- The picker and the card line: Institution #559.
- A catalog for a seat already running when the Fleet started: it is read at the seat's next start, and Configuration says so meanwhile.

## Related

#571 (epic) · #862 · #700 · Institution #559

Decision Record impact: aligned-with ADR-0019 (no Fleet list, no hidden default).

Live latest-open sweep: latest 20 open issues in `neo-agent-brain` at 20:07Z, plus a search for model/list and catalog; no equivalent (#690 and #330 are the harness-type catalog, closed).
A2A in-flight sweep: messages through 20:04Z; no claim on a model catalog.
MC sweep: "which models and reasoning efforts can a seat's harness offer; a declared model the account cannot use", 6 results, no prior decision. One adjacent note: the Claude desktop app's per-session setters are a later arm with their own admission, not v1.
Own-assignment sweep: 11 open in Brain, none overlapping beyond #862, which this leaf splits from.

Origin Session ID: 15ff44b9-9b0e-48b5-af34-9833bdfdf2f1
Retrieval Hint: "seat model catalog model/list supportedReasoningEfforts Start refusal"


## Timeline

- 2026-10-04T20:07:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-04T20:07:37Z @neo-opus-vega added the `enhancement` label
- 2026-10-04T20:07:37Z @neo-opus-vega added the `ai` label
- 2026-10-04T20:07:37Z @neo-opus-vega added the `agent-os` label
- 2026-10-04T20:07:39Z @neo-opus-vega added parent issue #571
- 2026-10-04T20:07:57Z @neo-opus-vega cross-referenced by #862
- 2026-10-04T20:08:33Z @neo-opus-vega cross-referenced by #559
- 2026-10-04T20:26:22Z @neo-opus-ada removed parent issue #571
- 2026-10-04T20:26:23Z @neo-opus-ada added parent issue #867
- 2026-10-04T20:34:51Z @neo-gpt-emmy cross-referenced by PR #866
- 2026-10-04T20:47:00Z @neo-opus-vega referenced in commit `1ec0937` - "feat(fleet): read a Codex seat's model catalog through its own app-server, every page, hidden entries flagged (#864)"
- 2026-10-04T20:47:00Z @neo-opus-vega referenced in commit `510e76c` - "feat(fleet): Start refuses a declared model its harness lacks, and Configuration can ask a seat's harness what it offers (#864)

seatModelCatalog reads per family: a Codex seat's app-server in its own logged-in home,
claude-code's effort levels from its own --help, nothing for the rest. Only a complete read
refuses, in Clio's words, before anything changes; the outcome rides the status as seatModel.
fleetSeatModelCatalog serves the picker: a stopped seat read fresh, a running one the catalog
its last start read. It is classed lifecycle-write, because the read starts an app-server in
the seat's home."
- 2026-10-04T20:55:24Z @neo-opus-vega cross-referenced by PR #869
- 2026-10-04T21:01:19Z @neo-opus-vega referenced in commit `a8003a0` - "refactor(fleet): where a harness reads a declared model is the shared catalog's seatSettings, one truth for Brain and Body (#864)

The launch contract map kept the fact in ai/, where the cockpit cannot read it. It moves to
src/fleet/contract/harnessTypes.mjs beside tenantMcpTarget, with resolveHarnessSeatSettings.
getHarnessSeatSettings and the launch map's three entries go."
- 2026-10-04T21:55:26Z @neo-opus-vega referenced in commit `6ccccce` - "fix(fleet): a seat's model refusal belongs to its latest start, so a later start never inherits it as its cause (#864)

Each provisioned start clears the seat's model record before its first refusal. A start refused at the Git
identity check, or one with the declaration withdrawn, no longer leaves an earlier "model is not available"
standing in the seat's status."
- 2026-10-04T21:55:26Z @neo-opus-vega referenced in commit `96c880c` - "fix(fleet): a Codex catalog read is over only when its app-server has exited, and an unreadable model list never proves a model missing (#864)

The answer waits for the child's exit: SIGTERM, then SIGKILL after a bounded grace, and a process that outlives
both hands back no catalog but stillRunning, since the seat's home is not free. A page is read whole or not at all:
a result without a data list, a row without an id or with unnamed efforts, or a non-string cursor reads as partial
(earlier pages kept) or unavailable, and nothing thrown while reading a message escapes the stream callback."
- 2026-10-04T21:55:26Z @neo-opus-vega referenced in commit `c00b632` - "fix(fleet): catalog reads and starts take a seat's home one at a time, so a running check never admits a second app-server (#864)

FleetManager.withSeatHome chains every operation on one seat's home: a stopped seat's picker read and a start's
provisioning wait for each other, and the running check runs inside the hold. A read queued behind a start finds
the seat running and answers the catalog that start kept. An operation that throws releases the home."
- 2026-10-04T21:55:26Z @neo-opus-vega referenced in commit `d9416c1` - "fix(fleet): every start reads and keeps its harness's catalog, so a seat started on defaults can be changed while it runs (#864)

The catalog is read at every provisioned start, declared or not, repo or not, and kept for Configuration; only a
declared value is checked against it. A read whose app-server outlived its signals refuses the start as
FLEET_SEAT_HOME_IN_USE. The model refusal says what did not happen (nothing cloned or configured, the harness not
started) instead of "nothing was changed", since the read itself runs the app-server in the seat's home."
- 2026-10-05T09:39:47Z @neo-opus-vega referenced in commit `27eba96` - "fix(fleet): a catalog app-server that outlives both signals keeps its seat's home until it exits, so no read or start runs beside it (#864)

The reader records the home and the server's pid when SIGTERM and SIGKILL both go unanswered, and clears it on that
process's exit. Until then every catalog read of that home answers that it is in use, naming the pid, without
starting a second app-server; the dispatcher asks before the login check, so the read each start runs first
refuses the start even when the home has lost its login meanwhile."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `0afe89c` - "feat(fleet): read a Codex seat's model catalog through its own app-server, every page, hidden entries flagged (#864)"
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `82771e0` - "feat(fleet): Start refuses a declared model its harness lacks, and Configuration can ask a seat's harness what it offers (#864)

seatModelCatalog reads per family: a Codex seat's app-server in its own logged-in home,
claude-code's effort levels from its own --help, nothing for the rest. Only a complete read
refuses, in Clio's words, before anything changes; the outcome rides the status as seatModel.
fleetSeatModelCatalog serves the picker: a stopped seat read fresh, a running one the catalog
its last start read. It is classed lifecycle-write, because the read starts an app-server in
the seat's home."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `92db496` - "refactor(fleet): where a harness reads a declared model is the shared catalog's seatSettings, one truth for Brain and Body (#864)

The launch contract map kept the fact in ai/, where the cockpit cannot read it. It moves to
src/fleet/contract/harnessTypes.mjs beside tenantMcpTarget, with resolveHarnessSeatSettings.
getHarnessSeatSettings and the launch map's three entries go."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `1115442` - "fix(fleet): a seat's model refusal belongs to its latest start, so a later start never inherits it as its cause (#864)

Each provisioned start clears the seat's model record before its first refusal. A start refused at the Git
identity check, or one with the declaration withdrawn, no longer leaves an earlier "model is not available"
standing in the seat's status."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `51319a6` - "fix(fleet): a Codex catalog read is over only when its app-server has exited, and an unreadable model list never proves a model missing (#864)

The answer waits for the child's exit: SIGTERM, then SIGKILL after a bounded grace, and a process that outlives
both hands back no catalog but stillRunning, since the seat's home is not free. A page is read whole or not at all:
a result without a data list, a row without an id or with unnamed efforts, or a non-string cursor reads as partial
(earlier pages kept) or unavailable, and nothing thrown while reading a message escapes the stream callback."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `f8976f3` - "fix(fleet): catalog reads and starts take a seat's home one at a time, so a running check never admits a second app-server (#864)

FleetManager.withSeatHome chains every operation on one seat's home: a stopped seat's picker read and a start's
provisioning wait for each other, and the running check runs inside the hold. A read queued behind a start finds
the seat running and answers the catalog that start kept. An operation that throws releases the home."
- 2026-10-05T09:43:45Z @neo-opus-vega referenced in commit `f59f496` - "fix(fleet): every start reads and keeps its harness's catalog, so a seat started on defaults can be changed while it runs (#864)

The catalog is read at every provisioned start, declared or not, repo or not, and kept for Configuration; only a
declared value is checked against it. A read whose app-server outlived its signals refuses the start as
FLEET_SEAT_HOME_IN_USE. The model refusal says what did not happen (nothing cloned or configured, the harness not
started) instead of "nothing was changed", since the read itself runs the app-server in the seat's home."
- 2026-10-05T09:43:46Z @neo-opus-vega referenced in commit `9de7a1c` - "fix(fleet): a catalog app-server that outlives both signals keeps its seat's home until it exits, so no read or start runs beside it (#864)

The reader records the home and the server's pid when SIGTERM and SIGKILL both go unanswered, and clears it on that
process's exit. Until then every catalog read of that home answers that it is in use, naming the pid, without
starting a second app-server; the dispatcher asks before the login check, so the read each start runs first
refuses the start even when the home has lost its login meanwhile."
- 2026-10-05T10:21:56Z @tobiu referenced in commit `f24815d` - "feat(fleet): Start refuses a declared model its harness lacks, and Configuration asks a seat's harness what it offers (#864) (#869)

* feat(fleet): read a Codex seat's model catalog through its own app-server, every page, hidden entries flagged (#864)

* feat(fleet): Start refuses a declared model its harness lacks, and Configuration can ask a seat's harness what it offers (#864)

seatModelCatalog reads per family: a Codex seat's app-server in its own logged-in home,
claude-code's effort levels from its own --help, nothing for the rest. Only a complete read
refuses, in Clio's words, before anything changes; the outcome rides the status as seatModel.
fleetSeatModelCatalog serves the picker: a stopped seat read fresh, a running one the catalog
its last start read. It is classed lifecycle-write, because the read starts an app-server in
the seat's home.

* refactor(fleet): where a harness reads a declared model is the shared catalog's seatSettings, one truth for Brain and Body (#864)

The launch contract map kept the fact in ai/, where the cockpit cannot read it. It moves to
src/fleet/contract/harnessTypes.mjs beside tenantMcpTarget, with resolveHarnessSeatSettings.
getHarnessSeatSettings and the launch map's three entries go.

* fix(fleet): a seat's model refusal belongs to its latest start, so a later start never inherits it as its cause (#864)

Each provisioned start clears the seat's model record before its first refusal. A start refused at the Git
identity check, or one with the declaration withdrawn, no longer leaves an earlier "model is not available"
standing in the seat's status.

* fix(fleet): a Codex catalog read is over only when its app-server has exited, and an unreadable model list never proves a model missing (#864)

The answer waits for the child's exit: SIGTERM, then SIGKILL after a bounded grace, and a process that outlives
both hands back no catalog but stillRunning, since the seat's home is not free. A page is read whole or not at all:
a result without a data list, a row without an id or with unnamed efforts, or a non-string cursor reads as partial
(earlier pages kept) or unavailable, and nothing thrown while reading a message escapes the stream callback.

* fix(fleet): catalog reads and starts take a seat's home one at a time, so a running check never admits a second app-server (#864)

FleetManager.withSeatHome chains every operation on one seat's home: a stopped seat's picker read and a start's
provisioning wait for each other, and the running check runs inside the hold. A read queued behind a start finds
the seat running and answers the catalog that start kept. An operation that throws releases the home.

* fix(fleet): every start reads and keeps its harness's catalog, so a seat started on defaults can be changed while it runs (#864)

The catalog is read at every provisioned start, declared or not, repo or not, and kept for Configuration; only a
declared value is checked against it. A read whose app-server outlived its signals refuses the start as
FLEET_SEAT_HOME_IN_USE. The model refusal says what did not happen (nothing cloned or configured, the harness not
started) instead of "nothing was changed", since the read itself runs the app-server in the seat's home.

* fix(fleet): a catalog app-server that outlives both signals keeps its seat's home until it exits, so no read or start runs beside it (#864)

The reader records the home and the server's pid when SIGTERM and SIGKILL both go unanswered, and clears it on that
process's exit. Until then every catalog read of that home answers that it is in use, naming the pid, without
starting a second app-server; the dispatcher asks before the login check, so the read each start runs first
refuses the start even when the home has lost its login meanwhile."
- 2026-10-05T10:21:57Z @tobiu closed this issue

