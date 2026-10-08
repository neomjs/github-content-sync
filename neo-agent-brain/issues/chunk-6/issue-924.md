---
id: 924
title: 'A moved seat''s memory lands in its own folder, whatever its harness'
state: OPEN
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees: []
createdAt: '2026-10-07T23:52:36Z'
updatedAt: '2026-10-07T23:52:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/924'
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
---
# A moved seat's memory lands in its own folder, whatever its harness

## Context

#571's predicate, amended 2026-10-08, requires a moved seat to read its own memory from `<seat>/memory` whatever its family, and to still read it after the harness's first native memory cycle and a cold restart. Today's import fails that for Codex and cannot serve OpenCode or Kimi.

- Euclid's Codex move lost 43 rollout summaries, and three indexes were rewritten, after the first native consolidation ([Sophie's controls](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6048122191)).
- OpenCode has a loader but no import path ([Emmy's intake](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6048380569)).
- [The owner decision](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6048895516) sets the shape below. [Its falsifier](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6049109501) passed on Codex CLI `0.162.0-alpha.2`: a 42,529-byte home `AGENTS.md` loaded whole, kept its bytes through the native sync, and loaded again after a cold restart.

Design authority: #571's terminal predicate and its owner decision 6048895516.

## The Problem

1. **The Codex import target is the producer's output folder.** `memoryDestination` returns `<CODEX_HOME>/memories`. Codex rebuilds that folder from its own database on the first turn, so imported files it does not index are deleted.
2. **The memory layer lives in three places.** Claude seats use `<agentsRoot>/<id>/memory`. OpenCode and Kimi scaffold theirs at `<instanceHome>/memory`, and `memoryDestination` returns `null` for them. Codex imports into the vendor folder. No single folder holds a seat's memory, and a harness change leaves it behind.
3. **The scaffold writes before the import.** Start prepares the workspace, then imports. Once a scaffolding family gets a destination, its birth files are already there, and the import refuses on the differing `MEMORY.md`.
4. **Any receipt skips the copy.** `importSeatMemory` copies only when no receipt exists, whatever destination the receipt names. Once Codex's destination moves, Euclid's seat refuses at Start with no way to re-consent, because `seatHoldsMemory` reads the receipt. If a scaffold filled the new folder first, it starts on a birth skeleton instead.
5. **Codex has no loader for a seat-owned layer.** `seatMemoryLayerTemplate` knows Kimi and OpenCode only. The file Codex loads at every start, `$CODEX_HOME/AGENTS.md`, already belongs to `projectSeatInstructions`. For a neo checkout that projection is `repository-supplied`, and its retire step removes a home file Fleet wrote.

## The Architectural Reality

Brain `dev` `197e659a`, folder `ai/services/fleet` (112 files in the structure map). No new file.

| File | Code LOC | Symbols |
|---|---|---|
| `seatMemoryImport.mjs` | 176 | `memoryDestination`, `importSeatMemory`, `seatHoldsMemory` |
| `projectSeatInstructions.mjs` | 73 | `HOME_INSTRUCTION_FILES`, `projectSeatInstructions`. Its comment records that Codex reads its home file whole, outside the project byte budget. |
| `prepareManagedAgentWorkspace.mjs` | 2219 | `prepareKimiArtifacts` and `prepareOpenCodeArtifacts` derive `memoryDir` from `instanceHome`; `prepareCodexArtifacts`; `convergeSeatInstructions`, `retireSeatInstructions` |
| `seatMemoryLayerTemplate.mjs` | 189 | `MEMORY_LAYER_BOOT_FILES`, `renderMemoryIndexMd` |
| `startAgentProvisioned.mjs` | 352 | preparation, then `importMemory` |
| `generateOpenCodeSeatConfig.mjs` | 215 | `external_directory` grants the seat home, the checkout and the runtime root |

## The Fix

1. **One destination.** `memoryDestination` returns `deriveAgentMemoryDir` for every family with a loader: `claude-code`, `claude-desktop`, `codex`, `codex-desktop`, `opencode` and `kimi-code`. Kimi and OpenCode preparation take `memoryDir` from it, and OpenCode's config grants that folder.
2. **Import, then scaffold.** The import converges before the layer's birth files are written. The scaffold stays create-only and fills only what the import did not bring.
3. **A receipt counts only for the destination it names, read relative to the seat folder.** A relocated seat, or one restored on another machine, keeps its receipt. A changed family destination counts as no receipt: the consented source is copied in as a first import, and the source stays the rollback. That is right for Codex, whose old destination holds generated state.
4. **Codex loads the layer through the one existing owner.** `projectSeatInstructions` renders the boot files from `<seat>/memory` into the home `AGENTS.md` at every Start, also when the checkout supplies the rules. Then the home file carries only the memory section. `seatMemoryLayerTemplate` gains the Codex branch.

The vendor's `memories/` and `[features] memories` stay as they are, outside acceptance.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `memoryDestination` | #571 predicate | `<agentsRoot>/<id>/memory` for every family with a loader | `null` for a family without one | its JSDoc | unit |
| the import receipt | owner decision, item 4 | counts when its destination, relative to the seat folder, equals the current one | a changed destination means a first import from the consented source | `importSeatMemory` JSDoc | unit |
| `$CODEX_HOME/AGENTS.md` | `projectSeatInstructions` as sole owner | carries the boot files, rendered at each Start | no layer, no memory section | the layer's about-file, Codex branch | unit; the falsifier above |
| OpenCode `external_directory` | the generator | grants `<seat>/memory` | none | generator JSDoc | unit |

## Acceptance Criteria

- [ ] `memoryDestination` returns `<agentsRoot>/<id>/memory` for all six families above. Kimi and OpenCode preparation use it, and OpenCode's config grants it.
- [ ] An OpenCode seat importing a folder that holds `MEMORY.md` starts with the source's bytes, and the scaffold adds only the missing birth files. Red first on today's order.
- [ ] A seat whose receipt names its Codex home's `memories` gets its consented source copied into `<seat>/memory` at the next Start, with no refusal and no birth skeleton. Red first. A control arm: a seat relocated to a new agents root keeps its receipt and is not re-imported.
- [ ] A Codex seat's home `AGENTS.md` carries the boot files from `<seat>/memory`, also when the checkout supplies `AGENTS.md`. A later Start re-renders after the boot files change, and no step removes a file Fleet did not write.
- [ ] The PR lists the FM-provisioned seats holding files at an old OpenCode or Kimi memory path. None is expected; any found is moved, not re-imported.
- [ ] **Post-merge, installed:** on a moved Codex seat, Euclid's first, the first turn loads a boot-file marker. After a native consolidation and a cold restart, the marker still loads and `<seat>/memory` is byte-unchanged. The source home stays intact.

## Out of Scope

- Which OpenCode sources a consent may name, such as Eos's seat-local memory. That is @neo-gpt-emmy's next step, under the importer's rule that a wire consent names only a recognized agent memory folder.
- Whether the Codex home file reloads after compaction. That needs its own witness.
- History and erasure recovery ([D#15702](https://github.com/orgs/neomjs/discussions/15702)), the portable bundle ([D#19455](https://github.com/orgs/neomjs/discussions/19455)), and model or effort settings (#923).

## Avoided Traps

- **Copying native producer state.** No inspected surface supports writing Codex's memory database, and a shared source home would carry unrelated chats.
- **A second writer for `$CODEX_HOME/AGENTS.md`.** The existing owner's retire step would delete it, refuse the Start, or report it as not Fleet's.
- **An absolute-path receipt check.** It would re-import over every relocated seat's own writes.
- **Re-copying generated files** to force equality after the seat has written its own memory.

## Related

#571 (parent) · #797 (the importer) · #898 (consent before the first Start) · #900 (relocated seat homes) · #812 (the same receipt principle for bootstrap effects) · #923 · neomjs/neo-agent-institution#12

Decision Record impact: none.

Live latest-open sweep: latest 20 open Brain issues at 2026-10-07T23:50Z, re-checked at 23:52Z; no equivalent. Org-wide open search for "memory import", "memoryDestination", "CODEX_HOME AGENTS.md" and "OpenCode memory" found only #571 and neomjs/neo-agent-institution#12.
A2A sweep: the latest 30 messages hold no claim on this scope.
MC sweep: the observed symptom; it surfaced Grace's D#15702 read, which this adopts, and the #797, #898 and #900 history; no prior decision against.
Own-assignment sweep: 15 open in this repository; #571 is the parent, and #142 (removal reconciliation) does not overlap.

unowned-rationale: one Brain PR, sized for a Codex builder. Overnight, Codex seats implement and Claude seats review; Emmy holds the adjacent OpenCode source step.

Origin Session ID: 4f4dd1c2-9db3-4ce5-8453-5fb675b918ed
Retrieval Hint: "seat memory destination deriveAgentMemoryDir Codex home AGENTS.md projectSeatInstructions import receipt relative"

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code

## Timeline

- 2026-10-07T23:52:37Z @neo-opus-ada added the `bug` label
- 2026-10-07T23:52:37Z @neo-opus-ada added the `ai` label
- 2026-10-07T23:52:37Z @neo-opus-ada added the `architecture` label
- 2026-10-07T23:52:37Z @neo-opus-ada added the `agent-os` label
- 2026-10-07T23:52:41Z @neo-opus-ada added parent issue #571

