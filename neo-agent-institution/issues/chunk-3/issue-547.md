---
id: 547
title: The setup card's tests run the pinned recipe through the real broker
state: CLOSED
labels:
  - enhancement
  - ai
  - tech-debt
  - testing
assignees:
  - neo-fable
createdAt: '2026-10-04T14:50:55Z'
updatedAt: '2026-10-04T18:31:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/547'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 535 The setup card opens with a guided front in the operator''s words'
  - '[ ] 540 The setup card offers a new witness attempt where the recipe names it'
closedAt: '2026-10-04T18:31:27Z'
milestone: FM v1
---
# The setup card's tests run the pinned recipe through the real broker

## Context

Row 1 of FM v1 (#351) has two card leaves waiting on the same test data. #540 needs the `verify` row's states (a refused write, a lost acknowledgement, the row fields of neomjs/neo-agent-brain#844) in the card's e2e. #535's fifth criterion asks that the normal-path goldens read the pinned Brain's recipe and presets. The design seat handed the fixture to the builder as its own leaf on 2026-10-04 (the frames on #535, [comment 5981064896](https://github.com/neomjs/neo-agent-institution/issues/535#issuecomment-5981064896)). This leaf carves #535's fifth criterion: the data source moves here, the goldens' re-capture stays there. No scope is added.

## The Problem

`test/playwright/fixture/setupRecipeSample.mjs` is a hand-written stand-in for the recipe, used by the card's unit spec, its e2e and the visual goldens. Its header says "nothing here is a hand-written status the renderer could mistake for truth". The file contradicts that:

- its step list has eleven rows and no `verify`; the pinned recipe has twelve;
- its page-side shell mutates `status` and `reason` by hand on every consent and effect;
- its placement says "no recorded quality floor" for a preset whose floor the pinned Brain records (the stranger read's third finding, [#351 comment 5979354380](https://github.com/neomjs/neo-agent-institution/issues/351#issuecomment-5979354380)).

So three test tiers pass against a recipe nobody evaluates. The card's two known defects (a top row whose `run` answers nothing; a `run` that never writes after a refused write) were found by a probe of the real broker, never by these tiers.

## The Architectural Reality

- `harness/setupBroker.mjs` `createSetupBroker` returns one handler per setup channel and runs under Node: `test/playwright/unit/harness/setupBroker.spec.mjs` already drives it over the pinned Brain (`node_modules/neo-agent-brain`) with `pinnedHost`, which replaces the command runner, the plane and the observers' three outside reads.
- The app reaches the shell through `globalThis.neoShell[name](request)` on the main thread (`src/main/addon/ShellPlane.mjs:12`), so a page-side `neoShell` whose methods call a function the test process exposes reaches a broker in that process.
- Consumers of the sample: `test/playwright/unit/apps/agentos/view/setup/createContainer.spec.mjs`, `test/playwright/e2e/agentos/FleetSetupCard.spec.mjs`, `test/playwright/visual/FleetCockpitVisual.spec.mjs`.

## The Fix

- `pinnedHost` moves from the broker's unit spec into `test/playwright/fixture/` and becomes the one scripted host: a temp state root, a recording command runner, an in-memory plane with its two switches (the recall lands, the write is refused), the production observers with the probe, the healthcheck and the validation injected.
- A page installer beside it exposes that broker to a page and defines `window.neoShell` over it: every evaluation, consent and effect the card asks for is the pinned recipe's answer. The credential channel keeps a temp file's path; no value enters the page.
- The card's e2e and unit specs read it. Adverse cases are named hosts: one that fits no local preset, one whose plane refuses the witness.
- The visual tier keeps the hand-written sample until #535 re-captures its goldens, and #535 deletes the file.

## Acceptance Criteria

- [ ] One scripted-host fixture under `test/playwright/fixture/` serves the broker's unit spec and the card's e2e; the broker's unit spec no longer defines its own.
- [ ] The card's e2e arms run against the real broker over the pinned Brain: the cold card, the consents and the first effect. No row status or reason is written by the fixture. (Re-scoped 2026-10-04: the arm from a cold host to `done ok` cannot pass while a run the card starts has no target. It is the witness of #351's gap line 5981291794 and lands with that fix; until then the completed run's chrome stays on the recorded sample in its own describe.)
- [ ] The card's unit spec reads the cold evaluation, the presets, the probe and the placement from the pinned recipe over the scripted host. An arm that tests the projector on a state the host cannot script may construct that one row, in the recipe's shape, and says so.
- [ ] One real-broker arm asks as the card asks, with no target bound by hand, and expects `done ok`; it is an expected failure until a Create run binds its profile's target (added 2026-10-04 by the planner, #351 comment 5981959620).
- [ ] Running the fixture executes no command, writes only under its temp root, and needs no real credential; an arm asserts the recorded commands and the root.
- [ ] `setupRecipeSample.mjs` keeps two consumers, each with its retirement named: the e2e's completed-run describe (retires with the fix of #351's gap line 5981291794) and the visual tier (#535 deletes the file).

## Out of Scope

- The goldens' re-capture and the card's front (#535).
- The new-attempt exit on the card (#540).
- The installed walk (#534): a broker over a scripted host is never its pass.

## Avoided Traps

- Pre-computing a set of evaluations and replaying them in the page: it is the hand-written sample again, one step removed, and it covers only the paths someone thought of.
- A generated JSON file committed beside the specs: it drifts from the pin the day the pin moves.

## Related

#351 (the row's epic) · #535 (its fifth criterion, carved here) · #540 · #534 · #541 (where `pinnedHost` was written) · neomjs/neo-agent-brain#844

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 20 open issues here, created-descending, read at 14:50Z (#540 … #391), plus org-wide searches for `setupRecipeSample` and "fixture pinned Brain recipe setup card". Only #535 and #351 match, and they hold the criterion this carves.
- A2A: all read-states, the last 12 messages (14:00Z to 14:34Z). No competing claim on the setup card's tests.
- Memory Core: one raw-memories query on the symptom returned nothing relevant.
- Own-assignment: #535, #540, #534, #475 read.
- Structure map: N/A, test fixtures only.

Decision Record impact: `none`.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "setup card e2e real broker pinned Brain scripted host fixture setupRecipeSample pinnedHost"




## Timeline

- 2026-10-04T14:50:55Z @neo-fable assigned to @neo-fable
- 2026-10-04T14:50:56Z @neo-fable added the `enhancement` label
- 2026-10-04T14:50:56Z @neo-fable added the `ai` label
- 2026-10-04T14:50:57Z @neo-fable added the `tech-debt` label
- 2026-10-04T14:50:57Z @neo-fable added the `testing` label
- 2026-10-04T14:51:05Z @neo-fable added parent issue #351
- 2026-10-04T14:51:06Z @neo-fable marked this issue as blocking #540
- 2026-10-04T14:51:09Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T14:51:09Z @neo-fable marked this issue as blocking #535
- 2026-10-04T14:51:12Z @neo-fable cross-referenced by #535
- 2026-10-04T14:51:36Z @neo-fable cross-referenced by #351
- 2026-10-04T15:05:48Z @neo-fable referenced in commit `26b8db8` - "test(harness): the scripted plane is the profile's own, and an arm holds the fixture to its limits (#547)

The plane that comes up answers as the Brain's canonical local plane (the profile's compose file pins that id in its health checks), whatever target a run was bound to; nothing answers before the project is up. The pinned arms bind the profile's target. A new arm asserts that a run bound to another id reads served-plane failed in the recipe's words, that the one compose command is recorded and never run, and that files land under the run's temp roots."
- 2026-10-04T15:09:57Z @neo-fable referenced in commit `f3816d7` - "test(agentos): the setup card's e2e asks the real broker over the pinned recipe (#547)

The cold card, the consents and the first effect now run against the setup broker over the pinned Brain on a scripted host: twelve rows, the recipe's own verdicts and reasons, the credentials kept as paths under the run's root, no command asked for. The completed run's chrome stays on the recorded sample shell in its own describe until a run the card starts can reach done on the real broker."
- 2026-10-04T16:12:11Z @neo-fable referenced in commit `db96a06` - "test(agentos): the setup card's view spec reads the pinned recipe's cold run, presets, probe and placement (#547)

The hand-written sample's constants are gone from the unit tier: the cold evaluation, the preset table, the probe and the placement come from the setup broker over the pinned Brain on the scripted laptop host. Rows an arm constructs are written by id, so the recipe's order is the recipe's to change. The sample had drifted: twelve rows instead of eleven, a hosted preset that is recommended rather than a candidate, another chat model in the table."
- 2026-10-04T16:13:23Z @neo-fable cross-referenced by PR #549
- 2026-10-04T16:21:11Z @neo-fable referenced in commit `6128e4b` - "test(harness): the card's own cold request is an arm that fails until a run binds its profile's target (#547)

The arm asks the broker as the card does, with no target bound by hand, and expects done ok. It is annotated as an expected failure: today write-env cannot render the run's missing target. With the profile's target bound the same arm reaches done ok, so the annotation turns CI red on the day the fix arrives and has to be removed with it."
- 2026-10-04T16:22:30Z @neo-fable cross-referenced by #848
- 2026-10-04T16:22:47Z @neo-fable cross-referenced by #550
- 2026-10-04T17:08:04Z @neo-fable cross-referenced by #540
- 2026-10-04T18:31:26Z @tobiu referenced in commit `c6d9e33` - "test(agentos): the setup card's tests run the pinned recipe through the real broker (#547) (#549)

* test(harness): the pinned setup host is one shared fixture, with a page installer for the card's e2e (#547)

pinnedHost moves out of the broker's unit spec into test/playwright/fixture/pinnedSetupHost.mjs. The scripted plane now follows the run: nothing answers until the compose project is up, then the plane its env carrier declares. The placement probe answers from a named machine. installPinnedSetupShell exposes the broker to a page and defines window.neoShell over it, so the card's e2e can ask the pinned recipe itself.

* test(harness): the scripted plane is the profile's own, and an arm holds the fixture to its limits (#547)

The plane that comes up answers as the Brain's canonical local plane (the profile's compose file pins that id in its health checks), whatever target a run was bound to; nothing answers before the project is up. The pinned arms bind the profile's target. A new arm asserts that a run bound to another id reads served-plane failed in the recipe's words, that the one compose command is recorded and never run, and that files land under the run's temp roots.

* test(agentos): the setup card's e2e asks the real broker over the pinned recipe (#547)

The cold card, the consents and the first effect now run against the setup broker over the pinned Brain on a scripted host: twelve rows, the recipe's own verdicts and reasons, the credentials kept as paths under the run's root, no command asked for. The completed run's chrome stays on the recorded sample shell in its own describe until a run the card starts can reach done on the real broker.

* test(agentos): the setup card's view spec reads the pinned recipe's cold run, presets, probe and placement (#547)

The hand-written sample's constants are gone from the unit tier: the cold evaluation, the preset table, the probe and the placement come from the setup broker over the pinned Brain on the scripted laptop host. Rows an arm constructs are written by id, so the recipe's order is the recipe's to change. The sample had drifted: twelve rows instead of eleven, a hosted preset that is recommended rather than a candidate, another chat model in the table.

* test(harness): the card's own cold request is an arm that fails until a run binds its profile's target (#547)

The arm asks the broker as the card does, with no target bound by hand, and expects done ok. It is annotated as an expected failure: today write-env cannot render the run's missing target. With the profile's target bound the same arm reaches done ok, so the annotation turns CI red on the day the fix arrives and has to be removed with it."
- 2026-10-04T18:31:27Z @tobiu closed this issue
- 2026-10-04T19:08:55Z @neo-gpt cross-referenced by PR #555

