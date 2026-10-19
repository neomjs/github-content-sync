---
id: 1201
title: The Fleet exports its seat lifecycle-state vocabulary
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - architecture
  - agent-os
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-19T15:32:11Z'
updatedAt: '2026-10-19T18:49:00Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1201'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 732
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 749 The roster reads every Fleet lifecycle state, not three of six'
closedAt: '2026-10-19T18:49:00Z'
---
# The Fleet exports its seat lifecycle-state vocabulary

## Context

Vega's defect-note (2026-10-18 17:20Z, fingerprint `c4f50117c327bd4a`, promoted here): Brain and the Fleet Manager share no definition of a seat's lifecycle state. `exited` was Sunday's P0 (#1138 → #1139): a word boot recovery wrote and the FM roster rejected, which disabled Start on every boot. Institution #732 (blocking for FM v1) lists the shared vocabulary as its L4 lane — "one shared state enum, one contract test both repos run" — under the operator's direction there: notice problems earlier than an install shows them. Promotion ground: a triage decision on that lane; the first occurrence was a production P0, the sibling words are source-read below.

## The Problem

The words a seat record's `state` can hold live in one codec line, `ai/services/fleet/seatRuntimeRecord.mjs:158` (`running · stopping · stopped · exited · failed · unavailable`), while `FleetLifecycleService.mjs` writes them as literals in ten places (`:770 :1154 :2043 :2082 :2101 :2207 :2379 :2806`, `status()` `:1650`) and the cockpit row producer adds its own fallback (`fleetCockpitStatus.mjs:217`: `runtime.state ?? 'unknown'`). Nothing exports the list. The FM copied three words (`apps/agentos/util/SourceHealth.mjs:269–290`) and reads every other one as "lifecycle and runtime facts contradict" — so a word the Brain adds or revives reaches the FM at the install, the class #1138 belongs to.

Two inconsistencies the missing contract hides: the `failed` record written when `spawn` throws (`:1154`) carries `error` and no `failureReason`, so the FM's reason line has nothing to print; `exited` is a read-only legacy word whose retirement ("leaves with the next snapshot version") is documented only in the codec's JSDoc, where no client reads it.

## The Architectural Reality

- `src/fleet/contract/` is the Brain's public, dependency-free Fleet vocabulary (`index.mjs`), exported as `neo-agent-brain/fleet-contract`; siblings `harnessTypes.mjs`, `cockpit.mjs` (`FLEET_COCKPIT_SOURCES.runtime = 'fleet:runtimeStatus'`, the source the lifecycle row names), `wire.mjs`. `test/playwright/unit/src/fleet/contract/fleetContract.spec.mjs` walks the entry's module graph with acorn and rejects any runtime dependency — the vocabulary must be data.
- The Institution consumes the entry at the pinned Brain revision (`harness/brain.mjs:123 loadFleetRuntimeContracts`; its unit specs under `test/playwright/unit/harness/` import `neo-agent-brain/…` directly). The cockpit reads the row through `RosterRow.mapRosterRow` → `SourceHealth.mapFleetSessionHealth`.
- Structure map (`npm run ai:structure-map -- --files --loc`): `ai/services/fleet` owns the service and the codec; the contract folder is the sibling precedent for client-shared words. Structural pre-flight: sibling-file lift of `cockpit.mjs`, no new directory.

## The Fix

1. `src/fleet/contract/seatLifecycle.mjs` (new, exported from `index.mjs`): `SEAT_LIFECYCLE_STATES` — the frozen words the Fleet writes (`running`, `stopping`, `stopped`, `failed`, `unavailable`); `SEAT_LIFECYCLE_READ_ONLY_STATES` (`exited`: read by the codec, written by no build since #1139, retired with the next snapshot version); `SEAT_LIFECYCLE_ROW_FALLBACK` (`unknown`: the cockpit producer's word for a supervised row without a state). One JSDoc line per word: what it means, whether the seat is off, which control a cockpit offers — `running` → Stop; `stopping` → Stop stays, no Start (Institution #725 Fix 3); `stopped` / `failed` / `unavailable` → Start, with `failureReason` shown; `unknown` → rejected evidence, no control.
2. `seatRuntimeRecord.mjs:158` reads the two lists instead of its literal (the deletion); `fleetCockpitStatus.mjs:217` reads the fallback constant.
3. `FleetLifecycleService.mjs:1154`: the spawn-failure record carries `failureReason` (the contract's `failed` obligation), `error` kept.
4. `test/playwright/unit/src/fleet/contract/seatLifecycle.spec.mjs`, should-first: the exported tuples equal the contract (a snapshot arm — red when a word is added without its contract row); every `state:` literal `FleetLifecycleService.mjs` writes is a member (an acorn scan, as `fleetContract.spec.mjs` does for imports); the codec accepts exactly states ∪ read-only; the dependency-free walk stays green with the new module.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-brain/fleet-contract` → `SEAT_LIFECYCLE_STATES`, `SEAT_LIFECYCLE_READ_ONLY_STATES`, `SEAT_LIFECYCLE_ROW_FALLBACK` (new) | `seatRuntimeRecord.mjs:158`; `fleetCockpitStatus.mjs:215–219` | frozen data, one JSDoc line per word incl. its control obligation | additive; absent in older pins | the module's JSDoc; `index.mjs` | AC-1 |
| `isSeatRuntimeSnapshot` (`seatRuntimeRecord.mjs:155`) | its JSDoc (`exited` read-only) | accepts states ∪ read-only, read from the contract | the same set as today | same | AC-2 |
| `FleetLifecycleService` record writes (`:770 :1154 :2043 :2082 :2101 :2207 :2379 :2806`) | `status()` `:1650` | every written word ∈ `SEAT_LIFECYCLE_STATES`; the `:1154` record gains `failureReason` | — | — | AC-3, AC-4 |
| cockpit row `lifecycle.state` | `fleetCockpitStatus.mjs:217` | `runtime.state ?? SEAT_LIFECYCLE_ROW_FALLBACK` | the same word as today | the row's JSDoc | AC-4 |

Decision Record impact: none — no ADR governs the roster vocabulary; the entry's own module JSDoc ("public, dependency-free Fleet vocabulary … shared by clients and the Brain") is the authority, and ADR 0026's 2026-08-08 fold names `src/fleet/contract/wire.mjs` as that client surface.

## Acceptance Criteria

- [ ] AC-1 `neo-agent-brain/fleet-contract` exports the three constants with per-word JSDoc; `fleetContract.spec.mjs` (the dependency walk) stays green.
- [ ] AC-2 `seatRuntimeRecord.mjs` holds no literal state list; the codec arm proves it accepts exactly states ∪ read-only and refuses a sixth written word.
- [ ] AC-3 the acorn arm proves every `state:` literal written by `FleetLifecycleService.mjs` is in `SEAT_LIFECYCLE_STATES`, red first against a planted literal.
- [ ] AC-4 the spawn-failure record (`:1154`) carries `failureReason`, and a cockpit row for it prints that reason (the existing `fleetCockpitStatus` arm extended).
- [ ] AC-5 (post-merge, owned by the Institution leaf) the FM's `sourceHealth.spec` imports the contract at the next Brain pin bump and maps every word.

## Out of Scope

The FM-side mapping (`SourceHealth`, `FleetStartPlan`, the card words) — the Institution leaf filed with this one. Retiring `exited` from the codec (its own snapshot-version bump). The launch-admission vocabulary (`launchAdmission.mjs`, already in the contract). #212 / #213's domain cut — this is the one-file slice of the Fleet domain's public surface Clio's #732 board calls "a #193-shaped slice".

## Avoided Traps

- Putting the list under `ai/services/fleet/` and importing it from the Institution: the runtime tree is not client-safe, and the contract spec's walk rejects it.
- A `describe` / `switch` helper instead of data: the FM must stay free to map the words onto its own display vocabulary (`CARD-CONTRACT.md`); the contract says what the words are and oblige, not how to draw them.
- Treating `unknown` as a state that offers Start: a supervised row without a word is rejected evidence; the fallback stays the producer's and the FM keeps refusing it.

## Related

Institution #732 (parent; the L4 row of its goal board) · #1138 / #1139 (`exited`) · Institution #725 (`stopping` kept Stop; cleanup `failed` mapped) · Institution #477 (every surface names its state with a reason) · #212 / #213 · the Institution leaf (filed with this one, blocked by this one).

Live latest-open sweep: the latest 20 open Brain issues at 2026-10-19T15:31Z — no equivalent; nearest #1180 (a different record shape). A2A claim sweep: the last 30 messages — no claim on the vocabulary; Clio's board row has been open since 10-18 23:15Z. Memory Core rationale sweep: `query_raw_memories` on the symptom returned Vega's 10-18 17:22Z turn — the Brain-only fix for #1138 was deliberate ("the codec keeps reading `exited`… rejecting it would freeze custody for every seat"), which this ticket keeps. Own-assignment sweep: none of my open tickets touches the Fleet contract. Structure map: `ai/services/fleet` (row 2383); the contract folder as cited.

Origin Session ID: 8c7b8590-c6ad-435f-ae38-9ab56e9597f2
Retrieval Hint: "seat lifecycle-state vocabulary fleet-contract SourceHealth contradict"

— Mnemosyne (Claude Fable 5.1, Claude Code)

## Timeline

- 2026-10-19T15:32:12Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-19T15:32:13Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-19T15:32:13Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-19T15:32:13Z @neo-fable-mnemosyne added the `testing` label
- 2026-10-19T15:32:13Z @neo-fable-mnemosyne added the `architecture` label
- 2026-10-19T15:32:14Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-10-19T15:32:54Z @neo-fable-mnemosyne cross-referenced by #749
- 2026-10-19T15:32:57Z @neo-fable-mnemosyne added parent issue #732
- 2026-10-19T15:33:35Z @neo-fable-mnemosyne marked this issue as blocking #749
- 2026-10-19T15:42:03Z @neo-fable-mnemosyne cross-referenced by PR #1202
- 2026-10-19T18:49:00Z @tobiu referenced in commit `2a69eab` - "feat(fleet): the Fleet exports its seat lifecycle-state vocabulary (#1201) (#1202)"
- 2026-10-19T18:49:01Z @tobiu closed this issue

