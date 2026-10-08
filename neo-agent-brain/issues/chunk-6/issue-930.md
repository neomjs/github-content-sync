---
id: 930
title: Let a managed seat explicitly select its own Codex memory
state: OPEN
labels:
  - enhancement
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-08T06:29:51Z'
updatedAt: '2026-10-08T06:37:30Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/930'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 571
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 924 A moved seat''s memory lands in its own folder, whatever its harness'
blocking:
  - '[ ] 603 Show a managed seat''s own memory in its existing chooser'
---
# Let a managed seat explicitly select its own Codex memory

## Context

[Sophie's candidate preflight](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6053260621) found an existing managed Codex seat with nonempty native memory output, no recorded import choice/receipt, and no shared memory folder. Another managed seat has a source consent and a legacy receipt; these are different migration cases.

Design authority: #571's selected-memory migration outcome and [its owner decision](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6048895516). The shared-folder work in #924 deliberately excludes additional recognized sources. This leaf supplies the observed missing source, without extending the history/portable-bundle designs.

## The Problem

The published importer accepts classic home profiles but rejects the managed Codex native-output path. A direct synthetic control on the current source accepts `~/.codex/memories`, rejects a derived managed `CODEX_HOME/memories` and `~/.ssh`, and returns `{state:'none'}` for absent consent. Discovery cannot offer the managed source, so an already-existing seat can reach the new scaffold without selecting its prior notes.

This is a missing supported migration path, not permission to copy every directory beneath an agents root or to infer consent from file presence.

## The Architectural Reality

- `seatMemoryImport.mjs`: `detectMemoryCandidates`, `normalizeMemoryImport`, `importSeatMemory` own metadata discovery, admitted source shapes and copying.
- `FleetControlBridge.fleetMemoryCandidates()` currently delegates a host-global, no-argument read. `devFleetServer.mjs` wires that host source; the plane without seat storage leaves it unavailable.
- `FleetRegistryService.configureAgent()` normalizes consent; the bridge serializes it with Start through `withSeatHome` and refuses running/already-imported seats.
- The registry definition and lifecycle's resolved instance root own seat identity and placement. `deriveAgentInstanceHome` / `deriveCodexHome` own the managed Codex source path.
- The existing Institution chooser has a selected binding but currently queries without its ID; its paired consumer leaf must forward that context.

Prescription checked: extend these owners, not a filesystem picker, second importer or view-side copy. Current structure map: `ai/services/fleet`, 112 files; no new module prescribed.

## The Fix

Support a closed optional `{id}` request on `fleetMemoryCandidates`. With an ID, resolve the existing seat and trusted placement on the host, retain existing classic candidates, and offer only that seat's derived native Codex memory directory when applicable and nonempty. Return `scope:{kind:'seat',id}` from the validated seat; a scoped request that cannot be answered is unavailable, never an empty host. No-argument Add Agent discovery retains its existing behavior.

Allow consent/import for that exact backend-derived same-seat source. The request never supplies an instance root or an arbitrary additional directory; another managed seat's directory and symlink escapes remain refused. Retain the existing copy verification and source rollback.

Before birth scaffolding, an existing managed Codex source with regular files, no chosen consent, and no established shared memory must require an explicit choice. A genuinely fresh seat remains fresh; explicit `none` remains the deliberate empty-memory option. Discovery is metadata-only; copying occurs only through the stopped-seat/Start contract.

## Contract Ledger

| Surface | Authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| `fleetMemoryCandidates({id?})` | Host registry + lifecycle placement | Seat-scoped metadata with exact `scope` echo; no-arg host discovery unchanged | Unknown/unavailable scope is unavailable, not empty | Bridge/source JSDoc | Wire/source tests, unknown ID and unwired host |
| Managed source normalization | Existing seat + derivation helpers | Accept only the selected seat's derived native Codex memory source | Other seats, arbitrary paths and symlinks refused | Importer JSDoc, OwnAgentTeam | Synthetic positive/escape controls |
| `configureAgent({id,memoryImport})` and Start | Existing consent/seat-home queue | Explicit choice before copying or birth files; preserve source and authored destination | Running/closed choice and conflicts refuse | Existing recovery contract | Production composer + temporary filesystem |
| Scoped result marker | Validated backend target | `scope:{kind:'seat',id}` | Consumer treats missing/mismatched scope as unavailable | Method result contract | Old-producer and mismatch controls |

## Acceptance Criteria

- [ ] A valid scoped read offers the selected managed Codex/ Codex Desktop source as metadata only; it does not enumerate other managed seats or read document contents.
- [ ] Unknown IDs, malformed payloads, unavailable host sources and mismatched placement never become a successful empty result. No-argument discovery is unchanged.
- [ ] The same-seat source can be explicitly consented and imported into the shared folder; original bytes remain intact. Another seat's path, arbitrary path and symlink escapes fail closed.
- [ ] Absent consent plus existing managed native notes and no shared memory refuses before scaffolding. Fresh/no-memory and explicit-`none` controls retain their intended behavior.
- [ ] Existing running/already-imported refusal, cancellation and no-overwrite controls still hold; no native database, session store or credential file is copied.
- [ ] Post-merge installed witness remains on #571 / Institution #12 after the paired chooser: select the source through the product, then marker → completed native cycle → unchanged shared-memory bytes → cold restart.

## Out of Scope

Arbitrary custom OpenCode/Kimi roots, portable bundles, native history/database restoration, memory merging/pruning, permission widening, automatic live-seat stops, and the Institution rendering change.

## Avoided Traps

An absent choice is not evidence that an old seat has no notes. A broad agents-root prefix is not same-seat authorization. A capability that ignores the requested ID is not scoped discovery. Source support is not installed retention.

## Related

#571 · #924 · #898 · #797. neomjs/neo-agent-institution#603 consumes this contract. Source work is blocked by #924's shared-folder implementation.

Decision Record impact: aligned with ADR 0041's explicit consent and evidence boundaries; no new credential or actor authority.

unowned-rationale: bounded follow-up to the measured candidate blocker, sequenced after #924; no overnight implementation claim.

Origin Session ID: 7cdef292-c073-447b-9afd-4eaab22ecdbf
Retrieval Hint: "managed Codex native memory absent consent same-seat source scoped candidates Sophie"

Live latest-open sweep: latest 20 open issues in Brain and Institution read at 2026-10-08T06:29Z with authors/labels/URLs; no equivalent. Latest 30 all-state A2A messages contain the source preflight but no competing claim. MC problem sweep recovered the shared-folder decision/audit; its final recency query was noisy and is not absence evidence. Own-assignment sweep: nine Brain issues and one Institution issue; the same-surface #924 body explicitly leaves additional source admission out, and Institution #42 is view-layer measurement, not this outcome. Current source and the direct normalizer control confirm the gap.


## Timeline

- 2026-10-08T06:29:52Z @neo-gpt-emmy added the `enhancement` label
- 2026-10-08T06:29:52Z @neo-gpt-emmy added the `ai` label
- 2026-10-08T06:29:52Z @neo-gpt-emmy added the `agent-os` label
- 2026-10-08T06:30:55Z @neo-gpt-emmy added parent issue #571
- 2026-10-08T06:30:57Z @neo-gpt-emmy marked this issue as being blocked by #924
- 2026-10-08T06:31:18Z @neo-gpt-emmy cross-referenced by #603
- 2026-10-08T06:31:46Z @neo-gpt-emmy marked this issue as blocking #603
- 2026-10-08T06:33:28Z @neo-gpt-emmy cross-referenced by #571
- 2026-10-08T06:46:33Z @neo-gpt-emmy cross-referenced by PR #928

