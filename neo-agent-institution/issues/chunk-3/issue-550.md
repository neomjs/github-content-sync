---
id: 550
title: A run the setup card starts takes the profile's target
state: OPEN
labels:
  - bug
  - ai
assignees:
  - neo-fable
createdAt: '2026-10-04T16:22:46Z'
updatedAt: '2026-10-04T17:50:24Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/550'
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
blockedBy:
  - '[x] 848 A Create run binds the target its profile declares'
blocking:
  - '[ ] 540 The setup card offers a new witness attempt where the recipe names it'
milestone: FM v1
---
# A run the setup card starts takes the profile's target

## Context

The Institution half of row 1's blocker (#351, gap line 5981291794; planner's disposition 5981959620 and 5981994832). The Brain half is neomjs/neo-agent-brain#848: `hostLayout()` carries the target the local profile declares. This leaf pins the Brain that carries it and makes the vessel's broker use it.

## The Problem

`src/main/addon/ShellPlane.mjs:120` forwards `target: data.target ?? null`, and the card never names one. `harness/setupBroker.mjs` `resolveRun` creates the run with `target: requested ?? {}`. The run has no plane id, no data root and no endpoint, so `write-env` fails and the run never reaches `done`. On the served cockpit over the real broker, `run` on `write-env` changes nothing and no word says why.

## The Architectural Reality

- `resolveRun` in `harness/setupBroker.mjs` is the one place a run is created or resumed; it already calls the Brain's `cli.hostLayout({stateRoot})` for every evaluation.
- The witness exists and is waiting: `test/playwright/unit/harness/setupBroker.spec.mjs`, "the card's own cold request, with no target bound by hand, reaches done ok", an expected failure (#547 / PR #549). With the profile's target bound it reaches `done ok`.
- The card's e2e keeps its completed-run arm on the hand-written sample shell until a run can complete on the real broker (`test/playwright/e2e/agentos/FleetSetupCard.spec.mjs`, second describe).

## The Fix

- The Institution pins the Brain commit on `dev` that carries neomjs/neo-agent-brain#844 and neomjs/neo-agent-brain#848 (`package.json`, the lock, `ci.yml`). The card's e2e reads the recipe's effect order from that pin (`write-secrets` first).
- `resolveRun` binds the layout's declared target when the request names none. A named target still wins; a resumed record keeps its own.
- The expected-failure arm becomes a plain arm.
- The card's e2e runs the completed run on the real broker (`pinnedSetupHost`), and the sample's e2e use is deleted.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `resolveRun` in `harness/setupBroker.mjs`, behind `setupEvaluate({target})` and every other channel (existing) | ADR-0034 §2.3 item 10; ADR 0041 §2.4; the Brain's `runTarget` (neomjs/neo-agent-brain#848) | a request that names no target: a new run is created already bound to `hostLayout({stateRoot}).target`; a run already bound keeps its binding; a named target wins. The broker decides none of it: it asks the Brain's `runTarget`, as the CLI does | a record created unbound before this change takes the profile's target on its next request, and its proof retires through the Brain's existing target-changed path | JSDoc on `resolveRun` | arms on the real broker in `setupBroker.spec.mjs`: the cold run (bound when created, an empty `history`), the hand-bound run, a later boot's resume; four source mutations |
| The Brain pin (`package.json`, the lock, `.github/workflows/ci.yml`) | this ticket's scope note | the Institution runs the Brain commit on `dev` that carries neomjs/neo-agent-brain#844 and neomjs/neo-agent-brain#848 | the one consumer change the pin alone forces is the effect order the card's e2e reads | none | every tier in CI |
| `pinnedSetupHost()` returns `profile`; its own `PROFILE_TARGET` constant is removed (test fixture) | this ticket | the fixture reads the profile's target from the pinned layout and states none itself | none | JSDoc | the specs that consume it |

## Acceptance Criteria

- [ ] A run created from a request without a target is bound to the layout's declared target; a request that names one and a resumed record are unchanged.
- [ ] `test.fail()` is removed from the cold-request arm and the arm passes: `done ok` over one witness write, with no target bound by hand.
- [ ] The card's e2e reaches the completed run on the real broker: the card retires, the chrome reads complete, the run's density is counted. `setupRecipeSample.mjs` has no e2e consumer left.
- [ ] No card file changes.
- [ ] The Brain pin carries neomjs/neo-agent-brain#844 and neomjs/neo-agent-brain#848, and every tier is green on it.

## Out of Scope

- The Brain's declaration (neomjs/neo-agent-brain#848).
- The installed proof: #534's second half, on a machine the operator chooses.
- The visual tier's sample (#535).

## Avoided Traps

- Stating the profile's plane id in the Institution: it would be a second declaration of what the Brain's layout states.
- Having the card send a target: the page would then choose which plane a host effect is bound to.

## Scope note (2026-10-04)

The pin rides in this leaf, as #524 carried its own (PR #543). The broker's call needs the Brain's `runTarget`, so the two cannot land apart in the other order, and a separate pin leaf would add a ticket, a review and a merge to a blocked row without separating any risk.

Measured before the pin exists, with the fixture's Brain root pointed at neomjs/neo-agent-brain#849's head `94d68578` (uncommitted):
- the new Brain with the old broker: unit 1395 of 1395; the card's e2e differs in one place, the effect order (`next: write-secrets`);
- with this change: unit 1394 of 1394, the card's e2e 2 of 2 with its run reaching `done` on the real broker (one command, one witness row), four source mutations of the broker change killed.

## Related

#351 (the row's epic) · neomjs/neo-agent-brain#848 (blocks this) · #547 / PR #549 · #534 · #540 · #535

Sweeps (2026-10-04):
- Live latest-open sweep: the latest 20 open issues here, created-descending, read at 16:22Z (#547 … #414). No equivalent.
- A2A: all read-states through 16:14Z. No competing claim.
- Memory Core: one raw-memories query on the symptom at 14:50Z returned nothing relevant; the decision record is the #351 thread.
- Own-assignment: #547, #540, #535, #534, #475 read.
- Structure map: N/A.

Decision Record impact: `aligned-with ADR 0041` §2.4.

Origin Session ID: 577754b6-3d27-48f5-911a-434605a54220
Retrieval Hint: "setup broker resolveRun target null layout target cold request done ok expected failure arm"



## Timeline

- 2026-10-04T16:22:46Z @neo-fable assigned to @neo-fable
- 2026-10-04T16:22:47Z @neo-fable added the `bug` label
- 2026-10-04T16:22:47Z @neo-fable added the `ai` label
- 2026-10-04T16:22:56Z @neo-fable added parent issue #351
- 2026-10-04T16:23:00Z @neo-fable marked this issue as being blocked by #848
- 2026-10-04T16:23:01Z @neo-fable added this to the **FM v1** milestone
- 2026-10-04T16:23:58Z @neo-fable cross-referenced by #351
- 2026-10-04T16:28:45Z @neo-fable cross-referenced by PR #849
- 2026-10-04T16:47:38Z @neo-fable-clio cross-referenced by #850
- 2026-10-04T17:33:44Z @neo-gpt-emmy cross-referenced by PR #852
- 2026-10-04T17:37:24Z @neo-fable cross-referenced by #540
- 2026-10-04T17:37:27Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-04T17:37:28Z @neo-fable marked this issue as blocking #540
- 2026-10-04T17:49:56Z @neo-fable cross-referenced by #848
- 2026-10-04T18:34:00Z @neo-fable cross-referenced by PR #555

