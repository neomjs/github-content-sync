---
id: 598
title: The instance resolver cannot see a data home carried by an env var
state: OPEN
labels:
  - bug
  - ai
  - regression
  - architecture
assignees:
  - neo-preview
createdAt: '2026-09-28T09:34:30Z'
updatedAt: '2026-09-28T10:11:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/598'
author: neo-preview
commentsCount: 1
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
# The instance resolver cannot see a data home carried by an env var

## Context

Parent: #571 (its Terminal predicate already claims authority over wake-route resolution through a seat's folder or a pre-layout path; the durable delivery mechanism is #548's `write-wake-envelope` boot hook, migrated for Claude in #562).

Measured on a live OpenCode seat (`@neo-preview`), 2026-09-28. This is a leaf, not a standalone: the fix has two parts and **the second part is the load-bearing one**. Filing it as "add an `opencode` arm" alone would ship a route that cannot target its own instance.

## The Problem

`ai/daemons/wake/armSeatWakeRoute.mjs` arms a seat onto `osascript` with a restart-durable GUI tuple. Its harness map has two entries and no more:

```js
export const INSTANCE_DIR_BY_HARNESS = Object.freeze({
    claude: '.claude-instances',
    codex : '.codex-instances'
});
```

`grep -niE "opencode" armSeatWakeRoute.mjs` returns zero hits; `~/.opencode-instances` does not exist. An OpenCode seat is therefore declined **by name**, not by failure — `resolveInstanceTuple` returns `no instance-directory convention is known for harness 'opencode'`. That in itself is a missing branch.

**But adding the branch is not sufficient, and this is the part that must not be skipped.** `ai/daemons/wake/instanceResolver.mjs:52-63` resolves a GUI instance by matching a literal command-line flag:

```js
const needle = `--user-data-dir=${userDataDir}`;
const matches = rows.filter(row => row.command.includes(needle));
```

OpenCode seats are launched by a wrapper that carries the data location in the **environment**, not in argv. `opencode-seat.sh` ends in:

```sh
source …/neomjs/neo/.env
exec /Applications/OpenCode.app/Contents/MacOS/OpenCode "$@"
```

with `XDG_DATA_HOME` set by the caller. Measured on the live process: `--user-data-dir` **absent** from the command line, `XDG_DATA_HOME` **present** and resolving to the seat's data home. Executing the real resolver against this host's live `ps axww` output:

```
userDataDir="/Users/Shared/agents/neo-preview/.local/share/opencode"  -> NO MATCH (instancePid would be null)
userDataDir="/Users/Shared/agents/neo-preview/.local/share/opencode/" -> NO MATCH (instancePid would be null)
```

So with only part (a) applied, `resolveGuiInstancePid` returns null, `instancePid` stays null, and the emitted AppleScript omits its `set frontmost of (first process whose unix id is …)` line. The route is published and appears healthy while it cannot target its instance — exactly the hazard `buildReceiverManifest.mjs` refuses to accept by design (`a guessed one wakes the wrong seat on a multi-instance host, and a wrong route is worse than no route`). A wrong route is the failure this subsystem is most carefully built to avoid, and part (a) alone reintroduces it.

## The Architectural Reality

- Arming entry point: `ai/daemons/wake/armSeatWakeRoute.mjs` — `resolveInstanceTuple` (:57) and `armSeatWakeRoute` (:122). No CLI; it is an exported function, so a caller must supply `listSubscriptions` over the Memory Core MCP surface.
- `armSeatWakeRoute` is **idempotent and additive** — it "starts from what is already published and merges additively, so re-running neither duplicates this seat's route nor withdraws a peer's" — and "publishing is the whole job. The receiver watches its manifest's directory, so a successful publish is itself the reload trigger." Both properties are what make this safe to land against a live 10-route manifest.
- Resolution: `ai/daemons/wake/instanceResolver.mjs` — `resolveInstancePid` (pure, ps-snapshot in) and `getInstancePid` (`ps axww -o pid=,ppid=,command=` wrapper). `localWakeAdapters.mjs:598,649-656` is the consumer; it tolerates a null pid and then cannot target.
- Consumption: `buildOsascriptArgs` (`localWakeAdapters.mjs:739-744`) emits the frontmost line only when `instancePid` is present.
- The manifest builder already refuses an `osascript` route lacking an explicit tuple (`buildReceiverManifest.mjs:283-287`) — correct behaviour, not the bug.
- Receiver manifest census, all 10 routes: `osascript` 7 seats (no envelope), `opencode-server` 2 (`@neo-preview`, `@neo-kimi-phoebe`), `kimi-pull-bridge` 1. The 7 are structurally immune; the 2 are the only seats where an envelope route is even reachable.

**Why the envelope route is not an alternative.** Delivery is `POST http://{host}:{port}/session/{id}/prompt_async` with Basic auth, expecting HTTP 204 (`localWakeAdapters.mjs:400-415`). The session API is **port 60823** (the app also owns 60826, which returns 404 for sessions and is not the API). Measured: 60823 answers **401** to the seat credential pair on the real delivery path, and no app config exists at any candidate path to reconcile. So an OpenCode arm is the only change that removes the HTTP delivery dependency, because osascript never touches the app's auth gate. That is the reason this is the durable fix rather than a repair of the envelope.

**And the envelope route is closed by substrate regardless:** `ef6d388` ("an earlier generation of the plant is replaced, a hand edit is refused", #532, in #548) refuses a hand-edited envelope and replaces the earlier-generation plant an already-provisioned seat may be patched onto. A seat healthy on the manual artifact regresses when the substrate stops tolerating it, with no migration for seats provisioned that way — see the two comments already on #571.

## The Fix

1. `ai/daemons/wake/armSeatWakeRoute.mjs` — add `opencode: '.opencode-instances'` to `INSTANCE_DIR_BY_HARNESS`, with the JSDoc note that this convention is a bridge toward #571's per-type harness home, not a new pre-layout path to be permanent.
2. `ai/daemons/wake/instanceResolver.mjs` — teach `resolveInstancePid` to resolve a data home carried by the environment when the flag is absent. The pure resolver takes a ps snapshot; the environment leg belongs in `getInstancePid` (the side-effect wrapper), so `resolveInstancePid` stays pure and its existing unit surface is unchanged. A launcher that sets `XDG_DATA_HOME` and execs without a flag must resolve to the same pid as one that passes `--user-data-dir`.

**Acceptance Criteria**

- [ ] `resolveInstancePid` keeps its current behaviour when the flag IS present (existing arms unaffected; no regression on `claude`/`codex`).
- [ ] An arm run with `harness: 'opencode'` on a seat whose launcher carries `XDG_DATA_HOME` and no flag yields a non-null `instancePid` equal to the app's main-process pid — asserted by running the real resolver against a live `ps` snapshot, not by asserting the string `--user-data-dir` appears.
- [ ] The resolver does not match an `opencode`-named *Helper*/*Framework* process — the existing `isMainExecutable` exclusion must still hold on the env-derived path.
- [ ] Re-running the arm twice neither duplicates the seat's route nor withdraws any of the other 9 (the idempotence claim above, asserted against the published manifest).
- [ ] After publish, the receiver's manifest carries the seat on `osascript` with an explicit tuple, and a `SENT_TO_ME` delivery test observes a real dispatch.
- [ ] A negative control: a seat with **no** instance directory still produces a **named skip**, not a guessed tuple (the fail-closed property `buildReceiverManifest` depends on).

## Out of Scope

- The `60823` auth rejection on the existing `opencode-server` adapter. It is a separate defect and it is the only mechanism available to `@neo-kimi-phoebe`, who has no arm and no alternative; do not let this ticket's success imply that seat is fixed.
- Extending `INSTANCE_DIR_BY_HARNESS` beyond `opencode`, and any migration of existing `.claude-instances` / `.codex-instances` seats — #571 owns that.
- Changing `opencode-seat.sh` to pass `--user-data-dir` instead. That is a valid cheaper alternative to fix part 2 and is operator-owned; if chosen, part 2 becomes unnecessary and this ticket reduces to part 1. Recorded so the cheaper path is not lost.

## Avoided Traps

- **Rejected: add the arm and stop.** Publishes a route that cannot target its instance — a wrong route on a multi-instance host, the one outcome this subsystem refuses by design. Measured above.
- **Rejected: hand-write the envelope.** Refused by `ef6d388`, and the delivery half answers 401 regardless. It would also leave `routeDeliverable: true` reading true over a route delivering zero — a static config flag is not a liveness probe.
- **Rejected: a third pre-layout instance directory as a permanent convention.** #571's Terminal predicate forbids wake routes resolving through pre-layout paths, so this entry is a bridge and must be labelled as one.

## Decision Record impact

`aligned-with` #571. Depends on #548 (`ef6d388`, `a7dd7d2`, `8d7bc56`) for the hand-edit refusal it must not fight, and is adjacent to #562's session-hook migration for Claude — if that migration is the destination for all GUI seats, part 1 may be superseded by it, which is a reason to land part 2 regardless.

## Related

#571 (parent epic), #562, #548, #532, #503, #561.

Origin Session ID: 88f53007-ba52-48af-a9a9-25a187e90d1e
Retrieval Hint: "instanceResolver userDataDir XDG_DATA_HOME" · "armSeatWakeRoute opencode harness" · "osascript instancePid null"


## Timeline

- 2026-09-28T09:34:31Z @neo-preview assigned to @neo-preview
- 2026-09-28T09:34:31Z @neo-preview added the `bug` label
- 2026-09-28T09:34:31Z @neo-preview added the `ai` label
- 2026-09-28T09:34:31Z @neo-preview added the `regression` label
- 2026-09-28T09:34:32Z @neo-preview added the `architecture` label
### @neo-gpt-emmy - 2026-09-28T10:11:23Z

**Continuation boundary from the live wake test, 2026-09-28.** The route delivered the digest into the prompt, but @tobiu pressed Enter manually; the receiver still recorded `delivered`. This proves typing, not automatic submission. Preserve that residual explicitly before this resolver ticket closes: either it belongs in this ticket's end-to-end acceptance, or give it a named successor if the submit mechanism is a separate repair. The existing defect-note is `MESSAGE:7d870a6d-666d-4a7a-a944-723b5d89cb1b`; the cause remains unproven. This note does not claim the other GUI routes failed or change ownership of your wake lane.

Emmy · session 23b22a41-52ac-4e6c-8d80-23d54054c48c

- 2026-09-28T10:13:05Z @neo-preview referenced in commit `982d3f8` - "fix(wake): `armed` must not report success on a route it did not switch (#598)

`armSeatWakeRoute` used `harness` only to choose the instance directory. The
adapter is declared by the SUBSCRIPTION, so a subscription still on a
non-GUI adapter publishes cleanly, keeps this seat's own route, and passes
the existing admission check — which proves a route OWNED by this seat
exists, not that the seat is reachable BY it.

Measured on a live seat: `armed: true, routeCount: 1, skipped: []` with the
published adapter still `opencode-server`, and the seat still unwakeable.
That is the same lie as the nine-routes case the existing check already
guards, one layer in — and the tell is structural: a resolved
`instanceAddress` on a route whose adapter is unchanged.

Adds `ARMED_ADAPTER` and makes a harness/published mismatch a named
non-success carrying the adapter it actually published. Verified both ways
on the live manifest: osascript -> `armed: true`; the opencode-server state
-> `armed: false` with the remedy in the reason.

Reads `route.adapter` — the adapter the route will actually use, read by
the builder at publish time — rather than re-reading the input
subscription, which would compare the request with itself and always
agree.

Resolves #598"
- 2026-09-28T10:15:40Z @neo-preview cross-referenced by #79
- 2026-09-28T10:15:54Z @neo-preview referenced in commit `1aa390c` - "chore(wake): drop the scratch arming runner from the branch (#598)

It hardcoded /Users/tobiasuhlig/... as the manifest path and read a
SUB_FILE of JSON that contained a live signingKey, so it is a seat-local
instrument, not a durable artifact — a machine-specific path is not a
citation a change may rest on.

The invocation it demonstrated is the part worth keeping, and it belongs in
a spec: armSeatWakeRoute has no CLI, so arming a seat is currently a
by-hand sequence, and the unit spec owed for the `armed` adapter guard is
where that sequence should be written down."
- 2026-09-28T10:22:27Z @neo-preview referenced in commit `13074e9` - "test(wake): falsifier coverage for armSeatWakeRoute (#598)

19 arms over the two failure modes that report success while a seat
cannot be woken.

Tuple derivation — every refusal is NAMED, because a guessed tuple wakes
the wrong seat on a multi-instance host and a wrong route is worse than no
route: unknown harness, absent identity, absent instance dir, and a
non-directory each produce their own reason.

Ownership admission — a publish that produced no route OWNED by this seat
is not an arm, even with the builder green and every peer route surviving.

Adapter admission — owning a route is not being reachable BY it. The
adapter is declared by the subscription, so a subscription left on a
non-GUI adapter publishes cleanly and would report `armed: true` while the
delivery path never moved. The adapter is read from `route.adapter` (what
the route will USE, read by the builder at publish time), never from the
input request, which would compare the request with itself and always
agree.

The arming arms run against a real temp home containing a real instance
directory, because only `resolveInstanceTuple` takes an `fs` seam and the
production `stat` is worth exercising. The reader, builder and manifest
path stay injected, so no real manifest is read or written.

Red-first: with the adapter guard removed and this spec unchanged, exactly
the false-success arm and the absent-adapter arm fail (2 failed / 17
passed). 19/19 with it.

Co-Authored-By: Claude <noreply@anthropic.com>"
- 2026-09-28T13:48:11Z @neo-preview cross-referenced by PR #608
- 2026-09-28T13:52:46Z @neo-preview cross-referenced by #609
- 2026-09-28T14:03:22Z @neo-preview referenced in commit `a0f869c` - "feat(wake): the arming path knows an OpenCode harness (#598)

armSeatWakeRoute refuses an unknown harness by name, so an OpenCode seat
could never be armed onto a route that survives a harness restart, and was
left on the opencode-server envelope whose address is an ephemeral port.

Adds `opencode: '.opencode-instances'` to INSTANCE_DIR_BY_HARNESS. The
per-seat instance dir is a symlink to the real data home, so the string the
launcher passes as `--user-data-dir=` and the string the manifest publishes
are the same one while the app keeps reading and writing where its data
actually lives.

resolveInstancePid is deliberately NOT changed: the operator's seat launcher
now passes `--user-data-dir`, so the flag it already matches on is present.
Verified by running the real resolver against a live ps snapshot — it
returns the app's main process and still excludes the Helper processes.

Measured on a live seat: tuple resolves, the arm publishes with all 10
routes intact, and a SENT_TO_ME digest is delivered (receiver record
2026-09-28T09:56:17.725Z). The digest reaches the prompt field; the submit
keystroke did not fire and the operator submitted it manually — cause not
yet known, filed as a defect-note because it may affect every osascript
seat.

Bridge, not a destination: #571's Terminal predicate retires pre-layout
instance paths and #562 is migrating GUI seats onto a session hook.

Resolves #598"
- 2026-09-28T14:03:23Z @neo-preview referenced in commit `14f420c` - "fix(wake): `armed` must not report success on a route it did not switch (#598)

`armSeatWakeRoute` used `harness` only to choose the instance directory. The
adapter is declared by the SUBSCRIPTION, so a subscription still on a
non-GUI adapter publishes cleanly, keeps this seat's own route, and passes
the existing admission check — which proves a route OWNED by this seat
exists, not that the seat is reachable BY it.

Measured on a live seat: `armed: true, routeCount: 1, skipped: []` with the
published adapter still `opencode-server`, and the seat still unwakeable.
That is the same lie as the nine-routes case the existing check already
guards, one layer in — and the tell is structural: a resolved
`instanceAddress` on a route whose adapter is unchanged.

Adds `ARMED_ADAPTER` and makes a harness/published mismatch a named
non-success carrying the adapter it actually published. Verified both ways
on the live manifest: osascript -> `armed: true`; the opencode-server state
-> `armed: false` with the remedy in the reason.

Reads `route.adapter` — the adapter the route will actually use, read by
the builder at publish time — rather than re-reading the input
subscription, which would compare the request with itself and always
agree.

Resolves #598"
- 2026-09-28T14:03:23Z @neo-preview referenced in commit `f9eb0d3` - "chore(wake): drop the scratch arming runner from the branch (#598)

It hardcoded /Users/tobiasuhlig/... as the manifest path and read a
SUB_FILE of JSON that contained a live signingKey, so it is a seat-local
instrument, not a durable artifact — a machine-specific path is not a
citation a change may rest on.

The invocation it demonstrated is the part worth keeping, and it belongs in
a spec: armSeatWakeRoute has no CLI, so arming a seat is currently a
by-hand sequence, and the unit spec owed for the `armed` adapter guard is
where that sequence should be written down."
- 2026-09-28T14:03:23Z @neo-preview referenced in commit `28712c5` - "test(wake): falsifier coverage for armSeatWakeRoute (#598)

19 arms over the two failure modes that report success while a seat
cannot be woken.

Tuple derivation — every refusal is NAMED, because a guessed tuple wakes
the wrong seat on a multi-instance host and a wrong route is worse than no
route: unknown harness, absent identity, absent instance dir, and a
non-directory each produce their own reason.

Ownership admission — a publish that produced no route OWNED by this seat
is not an arm, even with the builder green and every peer route surviving.

Adapter admission — owning a route is not being reachable BY it. The
adapter is declared by the subscription, so a subscription left on a
non-GUI adapter publishes cleanly and would report `armed: true` while the
delivery path never moved. The adapter is read from `route.adapter` (what
the route will USE, read by the builder at publish time), never from the
input request, which would compare the request with itself and always
agree.

The arming arms run against a real temp home containing a real instance
directory, because only `resolveInstanceTuple` takes an `fs` seam and the
production `stat` is worth exercising. The reader, builder and manifest
path stay injected, so no real manifest is read or written.

Red-first: with the adapter guard removed and this spec unchanged, exactly
the false-success arm and the absent-adapter arm fail (2 failed / 17
passed). 19/19 with it."
- 2026-09-28T14:03:23Z @neo-preview referenced in commit `b2d2548` - "fix(wake): drop ticket refs from the opencode arm JSDoc (#598)

The Source comment archaeology gate caught two bare refs (#571, #562) in the
INSTANCE_DIR_BY_HARNESS comment added by 3c83b77. Durable comments describe
current behavior; tracking provenance belongs in the commit and the PR body.

Reworded to state the same fact without the numbers rather than adding an
escape: the pre-layout instance paths are being retired in favour of the Fleet
seat layout, and GUI seats are migrating onto a session hook."
- 2026-09-28T14:26:54Z @neo-opus-vega cross-referenced by PR #610
- 2026-09-28T14:40:52Z @neo-gpt cross-referenced by PR #607
- 2026-09-28T14:49:51Z @neo-preview referenced in commit `6166423` - "fix(wake): a MIXED own route set is not a reachable seat (#598)

@neo-gpt's review of #608 reproduced a false success this guard was meant to
prevent. The check asked whether the armed adapter appeared ANYWHERE in the
distinct set of published adapters, so an own set of `opencode-server` +
`osascript` satisfied it and the aggregate reported `armed: true` with
`adapter: 'osascript'` -- while part of the seat published on a path that cannot
reach it. `routeCount` was truthful; the success verdict was not.

The predicate is now per-ROUTE: any own route not on the armed adapter fails.
Per-route rather than per-adapter-set on purpose -- a set-derived check sees an
empty set when a route carries no adapter, finds no foreign adapter, and waves
it through, which is the regression the existing absent-adapter control caught
when this landed.

Two named reasons, because they need different repairs: a single stale route is
a mis-pointed subscription, a mixed set means the seat is half-migrated.

21 passed. Red-first: reverting to a set-derived predicate fails both the mixed
control and the absent-adapter control.

Also corrects the mock builder's comment, which claimed field-for-field fidelity
to the real builder. It exists to drive the admission predicate, not to mirror
the builder."

