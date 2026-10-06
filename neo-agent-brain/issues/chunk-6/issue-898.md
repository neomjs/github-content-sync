---
id: 898
title: A never-started seat takes its memory consent through configureAgent
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-06T11:36:05Z'
updatedAt: '2026-10-06T13:29:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/898'
author: neo-opus-ada
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
closedAt: '2026-10-06T13:29:46Z'
---
# A never-started seat takes its memory consent through configureAgent

## Context

Gap 12 of #571, the peer move into Fleet Manager. Mnemosyne proposed it in [6015254991](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6015254991); the owner accepted it, with this shape, in [6015359685](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6015359685).

The operator's installed FM holds twelve seat rows, defined 09-30 to 10-05 on a build without a memory step (measured 10-06, names only, `brain/fleet/registry.json`). None of them carries `memoryImport`. The operator said on 10-06 that the names and PATs are already added, that peers move one by one, and that each peer must verifiably get its markdown memory.

## The Problem

A memory consent can only be given when a seat is born. A row defined without one is a fresh seat, and its first Start opens on an empty memory folder. Nothing refuses, because no consent was skipped. So on the next build, every one of the twelve pre-defined peers would start without its memory, silently.

The alternative, removing and re-adding each seat, re-enters twelve PATs and creates twelve new definitions: `removeAgent` deletes the stored PAT with the row.

## The Architectural Reality

Read at Brain `origin/dev` `1b69ef75`.

- **The consent is birth-only.** `FleetRegistryService.defineAgent` records `memoryImport` through `normalizeMemoryImport` (`ai/services/fleet/FleetRegistryService.mjs:458-460`). `configureAgent` allows `id, harnessType, mcpServers, mcpTarget, gitName, gitEmail, model, reasoningEffort` and rejects anything else (`:661`). `updateAgent` takes only `metadata` and `modelProvider` (`:598`).
- **Convergence already handles a late consent.** `importSeatMemory` (`ai/services/fleet/seatMemoryImport.mjs:264`) copies a consented source into the family's destination when the seat holds no import receipt (`.neo-fleet-seat-memory-import.json`, `:32`), proves it identical, and receipts it. A row without consent starts empty by design. Nothing in the copy depends on when the consent was recorded.
- **The cockpit reaches configuration through one curated verb.** `FleetControlBridge.configureAgent` (`ai/services/fleet/FleetControlBridge.mjs:535`) turns validation failures into `{status: 'rejected', reason}` for the Accounts and Agent Detail surfaces.

Design authority: #797, the #351 steward's call B (2026-10-03): "the import is a per-seat consent born in `defineAgent`, converged in the seat, and refused at Start when unconverged." This leaf widens "born in `defineAgent`" to "given before the seat's first convergence". Mnemosyne proposed the change, standing in for that steward on #351. The receipt keeps its meaning (provenance of the copy, never the seat's status), and so does the Start check.

## The Fix

- **The registry records.** `FleetRegistryService.configureAgent` accepts `memoryImport`: a source path or `'none'`, normalized by `normalizeMemoryImport` exactly as at definition, or `null` to withdraw a consent. It persists the value and reads no seat state.
- **The bridge decides.** `FleetControlBridge.configureAgent` decides whether a consent may still be given, inside the seat's home queue (`FleetManager.withSeatHome`), held across the decision and the registry write. A Start already holding the seat finishes first, and no Start can begin between the decision and the write. It refuses, as a controlled rejection naming the seat:
  - while the seat is running, even with no import receipt and an empty destination ("A seat's memory import is chosen before its first Start, and `<id>` is running.");
  - once the seat holds its memory: an import receipt in its harness home, or memory at its family's destination (`seatHoldsMemory`, the same truths `importSeatMemory` reads). The reason ends "... `<id>` already holds its memory."
- The next Start converges the consent through the existing `importSeatMemory`. No second copy path is added.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `FleetRegistryService.configureAgent` | #797 call B, widened here | accepts, normalizes and persists `memoryImport` (source, `'none'`, or `null`) | an invalid source and unknown fields still refuse, nothing written | `configureAgent` JSDoc | unit specs |
| `FleetControlBridge.configureAgent` | the curated configuration verb | decides inside `withSeatHome`, held across the decision and the write; refuses a running seat (even with no receipt or memory files) or one holding its memory | a refusal reaches the cockpit as `{status: 'rejected', code: 'FLEET_SEAT_MEMORY_IMPORT_CLOSED', reason}`; the row is unchanged | JSDoc `@param intent` | unit specs, incl. a Start held in the queue |
| `importSeatMemory` at Start | #797 | unchanged: a consent set after definition converges like one set at definition | — | — | a spec defines without consent, configures it, then converges |

Decision Record impact: none. #797's ticket-level frame is widened by the steward's stand-in, and no ADR records the birth-only rule.

## Acceptance Criteria

- [ ] AC-1: `configureAgent({id, memoryImport})` records a normalized consent on a seat that is not running, holds no import receipt, and whose destination holds no memory. The public definition reads it back.
- [ ] AC-2: The same call is a controlled rejection with `code: 'FLEET_SEAT_MEMORY_IMPORT_CLOSED'`, naming the seat and the reason, and the row is unchanged, when the seat is running (including running with an empty destination and no receipt), holds a receipt, or holds memory at its destination. A rejection a person can correct carries no such code.
- [ ] AC-3: `memoryImport: null` withdraws a consent under the same condition.
- [ ] AC-4: A seat defined without consent, then configured with one, converges at Start exactly as a seat born with it: source copied, identical, receipted. Its Start check still refuses a consented seat whose destination reads empty.
- [ ] AC-5: An invalid source (one `normalizeMemoryImport` refuses) is rejected with that reason, and nothing is written.

## Out of Scope

- The cockpit's "Memory" row in Agent Detail › Configuration › Seat: the Institution sibling leaf.
- Moving the installed FM's seat root (Gap 13 of #571).
- Re-importing into a seat that has started. Its memory is its own from then on.

## Avoided Traps

- **Re-adding each seat.** It works, but it costs the operator twelve PATs and forfeits the rows' identities. It's the shape the owner had chosen on 10-04, superseded by the operator's 10-06 input.
- **Keying "never started" on a registry flag.** The receipt and the destination are the truths `importSeatMemory` already reads. A second flag could disagree with them.
- **Deciding on the files alone.** Sophie's review probe on #899 found the race: a consent accepted while a Start held the seat landed after that Start had read the definition, so the seat launched empty under a recorded consent. Hence the queue and the running check.

## Related

Parent: #571 · #797 · #825 · neomjs/neo-agent-institution#521 · neomjs/neo-agent-institution#548

Structure map: `ai/services/fleet` (existing owner folder; no new file).

Sweeps (2026-10-06 11:35Z, immediately before filing):
- Live latest-open: the latest 20 open issues here (#571 to #896). No equivalent; #571 is the parent.
- A2A: the last 30 inbox messages, all read states. Mnemosyne proposed the gap and asked the owner to file; no lane-claim on this scope.
- MC sweep: one query on the problem's nouns ("pre-defined seats start with empty memory, no memoryImport consent on registry rows, re-add seats twelve PATs"), 6 results. The prior decision found is the owner's 10-04 call to re-add each seat, superseded in 6015359685.
- Own-assignment sweep: 16 open here; none touches the memory consent (#142 is repo-removal key cleanup).
- Decision records: no ADR names `memoryImport` or a memory consent.

Origin Session ID: 4095e966-9503-44c4-a759-5e7293dc3daa

Retrieval Hint: "pre-defined seats no memoryImport consent birth-only configureAgent memory import before first Start Gap 12"


## Timeline

- 2026-10-06T11:36:06Z @neo-opus-ada added the `enhancement` label
- 2026-10-06T11:36:07Z @neo-opus-ada added the `ai` label
- 2026-10-06T11:36:07Z @neo-opus-ada added the `agent-os` label
- 2026-10-06T11:36:07Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-06T11:36:17Z @neo-opus-ada added parent issue #571
- 2026-10-06T11:37:53Z @neo-opus-ada cross-referenced by #572
- 2026-10-06T11:46:04Z @neo-opus-ada referenced in commit `7731ee7` - "test(fleet): a seat holding its memory refuses a withdrawn consent too (#898)"
- 2026-10-06T11:46:56Z @neo-opus-ada cross-referenced by PR #899
- 2026-10-06T11:52:07Z @neo-opus-ada cross-referenced by #573
- 2026-10-06T11:53:55Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-06T12:05:05Z @neo-opus-vega cross-referenced by PR #574
- 2026-10-06T12:06:54Z @neo-opus-ada referenced in commit `bf68519` - "feat(fleet): a memory consent queues behind a Start holding the seat, and a running seat refuses it (#898)"
- 2026-10-06T12:10:33Z @neo-opus-ada referenced in commit `5ad7e76` - "docs(fleet): the memory consent's module summary names its late path (#898)"
- 2026-10-06T12:12:19Z @neo-opus-ada cross-referenced by #900
- 2026-10-06T12:33:38Z @neo-opus-ada referenced in commit `b3bb0bc` - "feat(fleet): a closed memory choice carries its code (#898)

Both late-consent refusals (the seat runs, or already holds its memory) now
carry code: 'FLEET_SEAT_MEMORY_IMPORT_CLOSED', in the bridge's own rejection
idiom. A consumer closes the choice on that code and keeps it open on any
other rejection, which a person can correct (Institution #574, RA-1)."
- 2026-10-06T12:44:40Z @neo-fable cross-referenced by #571
- 2026-10-06T13:29:47Z @tobiu referenced in commit `0b8477c` - "feat(fleet): a never-started seat takes its memory consent through configureAgent (#898) (#899)

* feat(fleet): a never-started seat takes its memory consent through configureAgent (#898)

A memory consent could only be given when a seat was defined, so a row added without one started
empty. configureAgent now accepts memoryImport (an agent's memory folder, 'none', or null to
withdraw), normalized as at definition. FleetControlBridge accepts it only while the seat holds no
memory yet: seatHoldsMemory reads the import receipt and the family's destination, the same truths
importSeatMemory reads at Start. The bridge's configureAgent is async for that read; the wire
dispatcher already awaits it.

* test(fleet): a seat holding its memory refuses a withdrawn consent too (#898)

* feat(fleet): a memory consent queues behind a Start holding the seat, and a running seat refuses it (#898)

* docs(fleet): the memory consent's module summary names its late path (#898)

* feat(fleet): a closed memory choice carries its code (#898)

Both late-consent refusals (the seat runs, or already holds its memory) now
carry code: 'FLEET_SEAT_MEMORY_IMPORT_CLOSED', in the bridge's own rejection
idiom. A consumer closes the choice on that code and keeps it open on any
other rejection, which a person can correct (Institution #574, RA-1)."
- 2026-10-06T13:29:47Z @tobiu closed this issue
- 2026-10-06T13:30:27Z @tobiu referenced in commit `64f518a` - "feat(agentos): Detail's Seat group offers a seat its memory import (#572) (#574)

* feat(agentos): Detail's Seat group offers a seat its memory import (#572)

A seat defined before Add Agent asked holds no memory consent, so its
first Start opens on empty memory. The Seat group gains a Memory row:
it names the recorded consent, and Choose/Change opens Add's own
choice (the Fleet's candidates in the same list, then Start with empty
memory). Save sends {id, memoryImport} through configureAgent
(neomjs/neo-agent-brain#898). The accepted readback re-seats the row,
and the Fleet's refusal for a seat that already holds its memory
stays on the row in its own words. ConfigIntentRoundTrip forwards
memoryImport and clears it when the readback omits it. Add's lead and
note now come from AddAgentFlow, so both surfaces ask in one set of
words.

* test(agentos): the Seat group's goldens carry the Memory row (#572)

The six seat-model goldens re-captured (the group grew by its Memory row), three new ones for the open choice in both skins and a recorded consent, a driver that opens the row on a fixture answer, and the baseline stamp. The full visual suite passes 42/42 against them.

* feat(agentos): the Fleet's refusal closes a seat's memory choice for good (#572)

#572's AC-3, as Ada rephrased it for option (b): once the Fleet
refuses a seat's consent (Brain #899 refuses a seat that already holds
its memory), the row keeps that refusal on its status line and offers
the choice no more. To tell that apart from failures,
ConfigIntentRoundTrip hands its sink a fourth argument,
{refused: true}, only when the Fleet itself answered "rejected". An
unreachable Fleet or an invalid answer leaves the choice open to try
again. Nothing reads the reason's wording.

* fix(agentos): the Memory row closes only on the Fleet's closed-choice code, says what was recorded, and names the chosen folder (#572)

Emmy's round 1 on #574, all three in place:

- RA-1: the round trip hands its sink the Fleet's code for a rejection, never a bare
  "refused". Only FLEET_SEAT_MEMORY_IMPORT_CLOSED (Brain #899: the seat runs or already
  holds its memory) withdraws the offer. Any other refusal keeps the choice and Save
  open, its words on the row.
- RA-2: the line says what is on record ("no import choice recorded", "recorded:
  import ..."), never what the seat's home holds, and never that an import ran.
- RA-3: Details under the choice names the chosen candidate's folder before Save, in one
  rule shared with Add Agent (AddAgentFlow.memoryDetail). Two candidates that read alike
  are told apart."

