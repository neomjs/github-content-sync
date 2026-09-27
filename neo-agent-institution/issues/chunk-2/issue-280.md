---
id: 280
title: 'Add agent registers a seat that already runs in its own harness: external, no PAT, never startable here'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-09-27T09:18:57Z'
updatedAt: '2026-09-27T12:21:25Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/280'
author: neo-opus-ada
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
closedAt: '2026-09-27T12:21:25Z'
---
# Add agent registers a seat that already runs in its own harness: external, no PAT, never startable here

## Context

The operator (2026-09-27, relayed by @neo-gpt-emmy) asked for the real team to be added through FM's Add agent. Every current peer runs in its own harness (Claude Desktop, Codex Desktop, …); none is launched by Fleet. Today the product has one registration path, and it makes Fleet the launcher:

- `AddAgentFlow.createDefineAgentIntent` stamps `launchOwner: 'fleet'` unconditionally (`apps/agentos/util/AddAgentFlow.mjs`, the intent literal).
- The shell prompts for a PAT on every `defineAgent` (`harness/fleetCapability.mjs`, the credential-method branch), and refuses an empty one.

Registered that way, a live peer becomes a seat Fleet may start. A Start from the cockpit, or the wire `startAgent`, would spawn a second instance of the same identity in a fresh managed clone.

Converged on A2A (Euclid, Emmy, Vega, Ada; the path and gates are recorded on #7, issuecomment-5854555659):
- existing seats register as `external`;
- `neomjs/neo-agent-brain#565` (Vega) makes an explicitly passed `launchOwner` stamp `launchOwnerSince` in the same write, so `launchRefusalOf` refuses a Start from birth.

## The Problem

- The ownership choice is hard-coded, and the wrong way round for the team that exists today.
- A PAT is demanded for a seat whose credentials Fleet never uses. The seat already authors through its own harness and env.
- **Cross-version hazard** (Euclid): the installed product pins Brain `c6c92c2`, which lacks #565. There, `launchRefusalOf` returns `null` for an `external` row without `launchOwnerSince`, and `defineAgent` writes none. An external option shipped against that pin persists a row the wire Start still admits. A readback check after the write would come too late.

## The Architectural Reality

- `apps/agentos/util/AddAgentFlow.mjs`: `createDefineAgentIntent` (the intent), `validateDefinePayload(payload, {credentialRequired})` (the PAT requirement), `validateReadback`.
- `apps/agentos/view/fleet/instances/AddAgentForm.mjs`: username, PAT field, harness chip row. A second form lives in `apps/agentos/view/accounts/Panel.mjs`; #245 consolidates them. Both go through `AddAgentFlow`, so the choice belongs there.
- `harness/fleetCapability.mjs`: `projectPublicAgentIntent` already carries `launchOwner` through; the credential-method branch calls `credentialProvider` before `send`.
- Brain: `FleetRegistryService.defineAgent` (`launchOwner` defaults `external`; `credential` optional), `src/fleet/contract/launchAuthority.mjs` `launchRefusalOf`, and #565.
- The Brain pin: `package.json` + lock + the CI ref (the #269 / #270 pattern).

## The Fix

1. The form asks who launches the seat. "It already runs in its own harness" gives `external` and is the default: a wrong external is undone by adopting the seat, while a wrong `fleet` can start a second copy of a live peer. "Fleet launches it" gives `fleet`. The choice rides the intent as an explicit `launchOwner`, never a constant.
2. External needs no PAT:
   - the credential field hides;
   - `validateDefinePayload` stops requiring one for `external`;
   - the shell forwards an `external` `defineAgent` as public intent without calling the credential provider.
   A `fleet` definition keeps the native credential custody exactly as today.
3. The same PR moves the Brain pin to a commit that carries #565, so the option never ships against a Brain that leaves the row startable.
4. The readback shows the truth: `launchOwner: 'external'` with `launchOwnerSince`, `launchRefusal` non-null, and no Start on the card.
5. An external seat is registered only through the Fleet a packaged shell started from its bundled Brain. That organism travels with its UI at one commit, so it carries the birth stamp. Three other paths fail closed, before the write:
   - a browser bridge may reach any Fleet server (`AddAgentFlow.canRegisterExternal`);
   - a shell may reuse an incumbent Fleet that answers on the same port, bearer and viewer;
   - a checkout shell spawns from an env-selected Brain root.
   The shell's boot receipt records which case it is (`bundledFleet`, from `harness/brain.mjs` `provesBundledFleet`), and `harness/fleetCapability.mjs` refuses an external define without it. Wire v1 carries no ownership semantics, so no negotiated fact could tell an older registry, which stores the row startable, from a current one. Euclid found the browser and reuse paths in review.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `AddAgentFlow.createDefineAgentIntent` (both forms) | this ticket | carries an explicit `launchOwner`, default `external`; a PAT crosses only for `fleet` in direct-browser mode | an unknown owner → `external` | JSDoc | `addAgentFlow.spec` AC-1 arm |
| `AddAgentFlow.validateDefinePayload` | this ticket | `external` needs no credential; `fleet` in direct-browser mode needs one; `external` needs `externalAllowed` | refuses before any write | JSDoc | `addAgentFlow.spec` AC-2 / AC-6 arms |
| `AddAgentFlow.canRegisterExternal` | Fix 5 | true only for the shell ingress | false ⇒ `external` refused | JSDoc | `addAgentFlow.spec` AC-6 arm |
| Shell credential custody (`harness/fleetCapability.mjs`) | this ticket | an `external` define forwards as public intent, with no provider call, and only when the boot receipt carries `bundledFleet`; `fleet` and an omitted owner keep the provider | no `bundledFleet` ⇒ a refused envelope before any write | JSDoc | `fleetCapability.spec` AC-3 / AC-6 arms |
| Boot receipt `bundledFleet` (`harness/brain.mjs` `provesBundledFleet`, `harness/main.mjs`) | Fix 5 | true only for a packaged shell that spawned its own Fleet | reuse or a checkout ⇒ false | JSDoc | `brain.spec` truth table |
| A refusal's text in the forms | Fix 5 | a `refused` answer to a request with no credential renders its reason; any other failure stays generic | — | JSDoc | `addAgentFlow.spec`, `Accounts.spec` |
| Rail form + Accounts form | this ticket | default `external`; PAT hidden and not required for it; the Accounts sample re-selects `external` | — | component JSDoc | `addAgentFlow.spec`, `Accounts.spec`, component witness, `accounts-config-surface` golden |
| Brain pin (`package.json`, lock, CI ref) | `neomjs/neo-agent-brain#566` | `66969169` | — | — | `fleetCapability.spec` AC-5 arm |
| Readback public fields | `FleetRegistryService.defineAgent` + `launchRefusalOf` | `launchOwner`, `launchOwnerSince`, and a non-null refusal for `external` | — | Brain JSDoc | AC-5 arm, red on `c6c92c2` |

## Acceptance Criteria

- [ ] AC-1: `createDefineAgentIntent` carries the chosen `launchOwner` explicitly for both choices; nothing hard-codes it (unit).
- [ ] AC-2: an `external` payload validates without a credential; a `fleet` payload in direct-browser mode still requires one (unit).
- [ ] AC-3: the shell forwards `defineAgent` with `launchOwner: 'external'` without calling the credential provider, and still prompts for `fleet` and for an intent without a `launchOwner` (unit on `fleetCapability`).
- [ ] AC-4: BOTH reachable forms render the choice, default to external, and hide the credential field for it: the rail's `AddAgentForm` (component arm) and the Accounts view's form (component arm; it is the reachable path in the installed saved layout, where the empty-roster CTA opens nothing). The Accounts golden is re-captured. Consolidating the two forms stays #245's.
- [ ] AC-5: the Brain pin carries #565. Against the pinned Brain, an explicit `external` define reads back `launchOwnerSince` and a non-null `launchRefusal` (unit arm over the pinned `FleetRegistryService` / `launchRefusalOf`).
- [ ] AC-6: no `external` seat is registered through a Fleet that cannot prove the birth stamp:
  - a browser bridge refuses in the flow and in both forms;
  - the shell refuses on a reused incumbent or a checkout Fleet;
  - each refusal comes before any write, and a `fleet` seat on the same path still writes (unit no-write arms).
  The packaged shell's own bundled Fleet registers one (AC-3, AC-5).

## Post-Merge Validation

- [ ] The pilot row in the installed shell, once rebuilt from a dev that carries this (proposed: Emmy, `codex-desktop`): `external` + `launchOwnerSince`, no Start, no PAT prompt, the presence band joins. It lands in the installed shell's own Fleet registry, a per-install launch overlay.

## Out of Scope

- The canonical plane registry. The composed plane still degrades `defineAgent` (S4, `neomjs/neo-agent-brain#52`) and `listAgents` / `fleetRoster` (S3, `neomjs/neo-agent-brain#53`). The team roster as an identity projection is #53's.
- Consolidating the two forms and the Accounts layout (#245).
- Adoption (`fleet`) for existing seats. That waits for the seat-home layout: Fleet's provisioned start makes a fresh clone and harness home, and each Claude seat's cwd-keyed memory would not follow.
- The empty-roster CTA that opens no form under a saved layout lacking the define-agent dock item (Emmy's finding; the Accounts view's form is the reachable entry until then).

## Related

#7 (the roster path record) · #245 · #10 · `neomjs/neo-agent-brain#565` (blocks this) · `neomjs/neo-agent-brain#52` · `neomjs/neo-agent-brain#53`

Live latest-open sweep: the latest 20 open issues of this repository at 2026-09-27T09:18:30Z, no equivalent (#245 is the form consolidation). A2A in-flight sweep (last 25, all read states): no claim on the ownership choice; #565's claim leaves "the UI half … an Institution leaf". Memory Core sweep ("Add agent form registers an existing peer … launch owner external"): only today's thread. Own-assignment sweep: #246 only.

Origin Session ID: f3d50317-fe3b-4773-b4ac-db05e1fa6812
Retrieval Hint: `query_raw_memories("Add agent external launchOwner no PAT existing peer launchOwnerSince cross-version pin")`




## Timeline

- 2026-09-27T09:18:57Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-27T09:18:59Z @neo-opus-ada added the `enhancement` label
- 2026-09-27T09:18:59Z @neo-opus-ada added the `agent-os` label
- 2026-09-27T09:18:59Z @neo-opus-ada added the `ai` label
- 2026-09-27T09:37:11Z @neo-gpt cross-referenced by PR #566
- 2026-09-27T09:46:58Z @neo-opus-ada cross-referenced by #571
- 2026-09-27T09:47:07Z @neo-gpt-emmy cross-referenced by #7
- 2026-09-27T09:57:43Z @neo-opus-ada referenced in commit `1962646` - "feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)

The operator asked for the real team to be added through Add agent, and
every current peer runs in its own harness. AddAgentFlow stamped
launchOwner 'fleet' on every definition and the shell asked for a PAT on
every defineAgent, so the only path enrolled a live peer as a seat Fleet
may start — a Start would spawn a second instance of it.

Both forms (the rail's AddAgentForm and the Accounts view) now ask who
launches the seat; "Runs in its own harness" is the default, because a
wrong external is undone by adopting the seat and a wrong fleet is not.
The choice always crosses as an explicit launchOwner. An external seat
needs no PAT: the field hides, validation stops asking, and the shell
forwards its public intent without calling the credential provider — a
Fleet-launched seat, and a define with no launch owner, keep the native
credential custody exactly as before (#171's first-Start guarantee)."
- 2026-09-27T09:57:43Z @neo-opus-ada referenced in commit `6618af2` - "build(deps): pin the Brain where an explicit launchOwner stamps launchOwnerSince (#280)

Brain c6c92c2 → 66969169 (the merge of neomjs/neo-agent-brain#566): package.json, lock and the
CI checkout ref move together. On the old pin an external row carried no launchOwnerSince, so
launchRefusalOf returned null and a wire Start was still admitted.

The new fleetCapability arm sends the shell-forwarded intents through the pinned Brain's own
registry, in a child process on that root's engine. The external row reads back
launchOwnerSince === createdAt with a non-null refusal; the Fleet-launched one stays startable.
Red on c6c92c2, green on 66969169.

The visual stamp is re-written for the rebased head; no golden changed."
- 2026-09-27T10:01:33Z @neo-opus-ada cross-referenced by PR #281
- 2026-09-27T10:13:42Z @neo-opus-ada referenced in commit `b9127bf` - "test(agentos): the credential journeys choose "Fleet launches it" before typing a PAT (#280)

The PAT field now shows only for a Fleet-launched seat, so the mounted credential-boundary
witness and the three add journeys that type a PAT pick that choice first; each keeps its own
premise. The component witness also asserts the field is hidden under the default."
- 2026-09-27T10:20:47Z @neo-opus-ada referenced in commit `9323ed7` - "feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)

The operator asked for the real team to be added through Add agent, and
every current peer runs in its own harness. AddAgentFlow stamped
launchOwner 'fleet' on every definition and the shell asked for a PAT on
every defineAgent, so the only path enrolled a live peer as a seat Fleet
may start — a Start would spawn a second instance of it.

Both forms (the rail's AddAgentForm and the Accounts view) now ask who
launches the seat; "Runs in its own harness" is the default, because a
wrong external is undone by adopting the seat and a wrong fleet is not.
The choice always crosses as an explicit launchOwner. An external seat
needs no PAT: the field hides, validation stops asking, and the shell
forwards its public intent without calling the credential provider — a
Fleet-launched seat, and a define with no launch owner, keep the native
credential custody exactly as before (#171's first-Start guarantee)."
- 2026-09-27T10:20:47Z @neo-opus-ada referenced in commit `804a26d` - "build(deps): pin the Brain where an explicit launchOwner stamps launchOwnerSince (#280)

Brain c6c92c2 → 66969169 (the merge of neomjs/neo-agent-brain#566): package.json, lock and the
CI checkout ref move together. On the old pin an external row carried no launchOwnerSince, so
launchRefusalOf returned null and a wire Start was still admitted.

The new fleetCapability arm sends the shell-forwarded intents through the pinned Brain's own
registry, in a child process on that root's engine. The external row reads back
launchOwnerSince === createdAt with a non-null refusal; the Fleet-launched one stays startable.
Red on c6c92c2, green on 66969169.

The visual stamp is re-written for the rebased head; no golden changed."
- 2026-09-27T10:20:47Z @neo-opus-ada referenced in commit `54c732c` - "test(agentos): the credential journeys choose "Fleet launches it" before typing a PAT (#280)

The PAT field now shows only for a Fleet-launched seat, so the mounted credential-boundary
witness and the three add journeys that type a PAT pick that choice first; each keeps its own
premise. The component witness also asserts the field is hidden under the default."
- 2026-09-27T10:20:47Z @neo-opus-ada referenced in commit `b105393` - "test(visual): re-stamp the baseline inputs on the rebased head (#280)"
- 2026-09-27T10:30:23Z @neo-opus-ada cross-referenced by #572
- 2026-09-27T10:55:08Z @neo-opus-ada referenced in commit `d88c08f` - "fix(agentos): only the shell registers a seat that runs in its own harness (#280)

A browser bridge may reach any Fleet server, and wire v1 reads the same on an older registry,
which stores an external row without `launchOwnerSince` and so startable, as on a current one.
`AddAgentFlow.canRegisterExternal` answers true only for the shell, whose Fleet is its own Brain
pinned at the birth stamp; `validateDefinePayload` refuses an external owner elsewhere before any
write, and both forms and the flow pass the answer. The wording says what the gate proves: Fleet
refuses to start the seat.

The Accounts sample is an existing peer: loading it re-selects external, hides and un-requires the
PAT field even after a switch to Fleet, and its status says what adding it takes from this ingress."
- 2026-09-27T10:59:16Z @neo-opus-ada referenced in commit `8eba9ea` - "feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)

The operator asked for the real team to be added through Add agent, and
every current peer runs in its own harness. AddAgentFlow stamped
launchOwner 'fleet' on every definition and the shell asked for a PAT on
every defineAgent, so the only path enrolled a live peer as a seat Fleet
may start — a Start would spawn a second instance of it.

Both forms (the rail's AddAgentForm and the Accounts view) now ask who
launches the seat; "Runs in its own harness" is the default, because a
wrong external is undone by adopting the seat and a wrong fleet is not.
The choice always crosses as an explicit launchOwner. An external seat
needs no PAT: the field hides, validation stops asking, and the shell
forwards its public intent without calling the credential provider — a
Fleet-launched seat, and a define with no launch owner, keep the native
credential custody exactly as before (#171's first-Start guarantee)."
- 2026-09-27T10:59:16Z @neo-opus-ada referenced in commit `68793e8` - "build(deps): pin the Brain where an explicit launchOwner stamps launchOwnerSince (#280)

Brain c6c92c2 → 66969169 (the merge of neomjs/neo-agent-brain#566): package.json, lock and the
CI checkout ref move together. On the old pin an external row carried no launchOwnerSince, so
launchRefusalOf returned null and a wire Start was still admitted.

The new fleetCapability arm sends the shell-forwarded intents through the pinned Brain's own
registry, in a child process on that root's engine. The external row reads back
launchOwnerSince === createdAt with a non-null refusal; the Fleet-launched one stays startable.
Red on c6c92c2, green on 66969169.

The visual stamp is re-written for the rebased head; no golden changed."
- 2026-09-27T10:59:16Z @neo-opus-ada referenced in commit `b63f21a` - "test(agentos): the credential journeys choose "Fleet launches it" before typing a PAT (#280)

The PAT field now shows only for a Fleet-launched seat, so the mounted credential-boundary
witness and the three add journeys that type a PAT pick that choice first; each keeps its own
premise. The component witness also asserts the field is hidden under the default."
- 2026-09-27T10:59:16Z @neo-opus-ada referenced in commit `e557b35` - "test(visual): re-stamp the baseline inputs on the rebased head (#280)"
- 2026-09-27T10:59:16Z @neo-opus-ada referenced in commit `3b6825c` - "fix(agentos): only the shell registers a seat that runs in its own harness (#280)

A browser bridge may reach any Fleet server, and wire v1 reads the same on an older registry,
which stores an external row without `launchOwnerSince` and so startable, as on a current one.
`AddAgentFlow.canRegisterExternal` answers true only for the shell, whose Fleet is its own Brain
pinned at the birth stamp; `validateDefinePayload` refuses an external owner elsewhere before any
write, and both forms and the flow pass the answer. The wording says what the gate proves: Fleet
refuses to start the seat.

The Accounts sample is an existing peer: loading it re-selects external, hides and un-requires the
PAT field even after a switch to Fleet, and its status says what adding it takes from this ingress."
- 2026-09-27T11:04:26Z @neo-opus-ada cross-referenced by PR #573
- 2026-09-27T11:24:31Z @neo-opus-ada referenced in commit `2e412e1` - "fix(agentos): the shell registers an external seat only through the Fleet it started from its bundled Brain (#280)

The shell ingress alone did not prove the Brain pin: a boot reuses an incumbent Fleet that
answers on the same port, bearer and viewer, and a checkout boot spawns from an env-selected
Brain root. Either may be an older registry that stores the external row startable.

`provesBundledFleet` answers true only for a packaged shell that spawned its own Fleet, whose
organism travels with its UI at one commit; the product boot's receipt carries it as
`bundledFleet`, and `fleetCapability` refuses an external define before the write without it.
A Fleet-launched seat still goes through. A refusal with no credential in the request now reads
as its reason in both forms; a PAT-bearing failure stays generic."
- 2026-09-27T11:32:23Z @neo-opus-ada referenced in commit `191fdff` - "feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)

The operator asked for the real team to be added through Add agent, and
every current peer runs in its own harness. AddAgentFlow stamped
launchOwner 'fleet' on every definition and the shell asked for a PAT on
every defineAgent, so the only path enrolled a live peer as a seat Fleet
may start — a Start would spawn a second instance of it.

Both forms (the rail's AddAgentForm and the Accounts view) now ask who
launches the seat; "Runs in its own harness" is the default, because a
wrong external is undone by adopting the seat and a wrong fleet is not.
The choice always crosses as an explicit launchOwner. An external seat
needs no PAT: the field hides, validation stops asking, and the shell
forwards its public intent without calling the credential provider — a
Fleet-launched seat, and a define with no launch owner, keep the native
credential custody exactly as before (#171's first-Start guarantee)."
- 2026-09-27T11:32:24Z @neo-opus-ada referenced in commit `1f68a90` - "build(deps): pin the Brain where an explicit launchOwner stamps launchOwnerSince (#280)

Brain c6c92c2 → 66969169 (the merge of neomjs/neo-agent-brain#566): package.json, lock and the
CI checkout ref move together. On the old pin an external row carried no launchOwnerSince, so
launchRefusalOf returned null and a wire Start was still admitted.

The new fleetCapability arm sends the shell-forwarded intents through the pinned Brain's own
registry, in a child process on that root's engine. The external row reads back
launchOwnerSince === createdAt with a non-null refusal; the Fleet-launched one stays startable.
Red on c6c92c2, green on 66969169.

The visual stamp is re-written for the rebased head; no golden changed."
- 2026-09-27T11:32:24Z @neo-opus-ada referenced in commit `698a2cb` - "test(agentos): the credential journeys choose "Fleet launches it" before typing a PAT (#280)

The PAT field now shows only for a Fleet-launched seat, so the mounted credential-boundary
witness and the three add journeys that type a PAT pick that choice first; each keeps its own
premise. The component witness also asserts the field is hidden under the default."
- 2026-09-27T11:32:24Z @neo-opus-ada referenced in commit `062faa4` - "fix(agentos): only the shell registers a seat that runs in its own harness (#280)

A browser bridge may reach any Fleet server, and wire v1 reads the same on an older registry,
which stores an external row without `launchOwnerSince` and so startable, as on a current one.
`AddAgentFlow.canRegisterExternal` answers true only for the shell, whose Fleet is its own Brain
pinned at the birth stamp; `validateDefinePayload` refuses an external owner elsewhere before any
write, and both forms and the flow pass the answer. The wording says what the gate proves: Fleet
refuses to start the seat.

The Accounts sample is an existing peer: loading it re-selects external, hides and un-requires the
PAT field even after a switch to Fleet, and its status says what adding it takes from this ingress."
- 2026-09-27T11:32:24Z @neo-opus-ada referenced in commit `6428e73` - "fix(agentos): the shell registers an external seat only through the Fleet it started from its bundled Brain (#280)

The shell ingress alone did not prove the Brain pin: a boot reuses an incumbent Fleet that
answers on the same port, bearer and viewer, and a checkout boot spawns from an env-selected
Brain root. Either may be an older registry that stores the external row startable.

`provesBundledFleet` answers true only for a packaged shell that spawned its own Fleet, whose
organism travels with its UI at one commit; the product boot's receipt carries it as
`bundledFleet`, and `fleetCapability` refuses an external define before the write without it.
A Fleet-launched seat still goes through. A refusal with no credential in the request now reads
as its reason in both forms; a PAT-bearing failure stays generic."
- 2026-09-27T11:32:25Z @neo-opus-ada referenced in commit `5db1a6f` - "test(visual): re-stamp the baseline inputs over the roster reconcile fix (#280)"
- 2026-09-27T12:12:24Z @neo-opus-ada referenced in commit `8192467` - "feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)

The operator asked for the real team to be added through Add agent, and
every current peer runs in its own harness. AddAgentFlow stamped
launchOwner 'fleet' on every definition and the shell asked for a PAT on
every defineAgent, so the only path enrolled a live peer as a seat Fleet
may start — a Start would spawn a second instance of it.

Both forms (the rail's AddAgentForm and the Accounts view) now ask who
launches the seat; "Runs in its own harness" is the default, because a
wrong external is undone by adopting the seat and a wrong fleet is not.
The choice always crosses as an explicit launchOwner. An external seat
needs no PAT: the field hides, validation stops asking, and the shell
forwards its public intent without calling the credential provider — a
Fleet-launched seat, and a define with no launch owner, keep the native
credential custody exactly as before (#171's first-Start guarantee)."
- 2026-09-27T12:12:24Z @neo-opus-ada referenced in commit `26ad383` - "build(deps): pin the Brain where an explicit launchOwner stamps launchOwnerSince (#280)

Brain c6c92c2 → 66969169 (the merge of neomjs/neo-agent-brain#566): package.json, lock and the
CI checkout ref move together. On the old pin an external row carried no launchOwnerSince, so
launchRefusalOf returned null and a wire Start was still admitted.

The new fleetCapability arm sends the shell-forwarded intents through the pinned Brain's own
registry, in a child process on that root's engine. The external row reads back
launchOwnerSince === createdAt with a non-null refusal; the Fleet-launched one stays startable.
Red on c6c92c2, green on 66969169.

The visual stamp is re-written for the rebased head; no golden changed."
- 2026-09-27T12:12:24Z @neo-opus-ada referenced in commit `b3fd46d` - "test(agentos): the credential journeys choose "Fleet launches it" before typing a PAT (#280)

The PAT field now shows only for a Fleet-launched seat, so the mounted credential-boundary
witness and the three add journeys that type a PAT pick that choice first; each keeps its own
premise. The component witness also asserts the field is hidden under the default."
- 2026-09-27T12:12:24Z @neo-opus-ada referenced in commit `01ce9e5` - "fix(agentos): only the shell registers a seat that runs in its own harness (#280)

A browser bridge may reach any Fleet server, and wire v1 reads the same on an older registry,
which stores an external row without `launchOwnerSince` and so startable, as on a current one.
`AddAgentFlow.canRegisterExternal` answers true only for the shell, whose Fleet is its own Brain
pinned at the birth stamp; `validateDefinePayload` refuses an external owner elsewhere before any
write, and both forms and the flow pass the answer. The wording says what the gate proves: Fleet
refuses to start the seat.

The Accounts sample is an existing peer: loading it re-selects external, hides and un-requires the
PAT field even after a switch to Fleet, and its status says what adding it takes from this ingress."
- 2026-09-27T12:12:25Z @neo-opus-ada referenced in commit `9328099` - "fix(agentos): the shell registers an external seat only through the Fleet it started from its bundled Brain (#280)

The shell ingress alone did not prove the Brain pin: a boot reuses an incumbent Fleet that
answers on the same port, bearer and viewer, and a checkout boot spawns from an env-selected
Brain root. Either may be an older registry that stores the external row startable.

`provesBundledFleet` answers true only for a packaged shell that spawned its own Fleet, whose
organism travels with its UI at one commit; the product boot's receipt carries it as
`bundledFleet`, and `fleetCapability` refuses an external define before the write without it.
A Fleet-launched seat still goes through. A refusal with no credential in the request now reads
as its reason in both forms; a PAT-bearing failure stays generic."
- 2026-09-27T12:12:25Z @neo-opus-ada referenced in commit `4560f98` - "test(visual): re-stamp the baseline inputs over the merged roster legend (#280)"
- 2026-09-27T12:21:25Z @tobiu referenced in commit `751b266` - "Merge pull request #281 from neomjs/ada/280-add-agent-external

feat(agentos): Add agent registers a seat that already runs in its own harness — external, no PAT (#280)"
- 2026-09-27T12:21:25Z @tobiu closed this issue
- 2026-09-27T12:22:04Z @neo-opus-ada cross-referenced by #287
- 2026-09-27T12:26:56Z @neo-opus-ada cross-referenced by #289
- 2026-09-27T12:42:33Z @neo-gpt cross-referenced by PR #290

