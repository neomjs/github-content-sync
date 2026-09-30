---
id: 644
title: Fleet seats in repositories without AGENTS.md load no instructions
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-30T17:14:47Z'
updatedAt: '2026-09-30T22:23:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/644'
author: neo-opus-grace
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
closedAt: '2026-09-30T22:23:31Z'
---
# Fleet seats in repositories without AGENTS.md load no instructions

## Context

Operator direction, 2026-09-30, recorded by @neo-gpt-emmy on neomjs/neo-agent-skills#100 ([issuecomment-5915607875](https://github.com/neomjs/neo-agent-skills/issues/100#issuecomment-5915607875)): the maintainer instructions leave the Engine, and each peer's home carries its instructions, generated from the Skills package. neomjs/neo-agent-skills#100 makes that composition importable as `neo-agent-skills/agents-md` (its PR is pending). This ticket is the caller.

**Sweep attestations (2026-09-30):** live latest-open checked (this repository, latest 20); the nearest are #642 (Codex context defaults) and #571 (seat folder layout), and neither writes instructions. The last 30 A2A messages carry no overlapping claim; the adjacent one is @neo-gpt-emmy's 16:15Z defect-note that a fresh Fleet seat lacks skills. The Memory Core sweep on the symptom found no prior decision. My own assigned issues here do not touch this surface. Structure map: `ai/services/fleet` holds 75 files and `prepareManagedAgentWorkspace.mjs` is 1,612 code lines, so the new logic gets a sibling module.

## The Problem

Measured on each repository's `dev`:

| repository | tracked `AGENTS.md` | tracked `.claude/CLAUDE.md` |
|---|---|---|
| `neomjs/neo` | 24,436 B | 24,436 B |
| `neomjs/devindex` | 24,272 B | absent |
| `neomjs/neo-agent-brain` | absent | absent |
| `neomjs/neo-agent-institution` | absent | absent |
| `neomjs/neo-agent-skills` | absent | absent |

Fleet writes no instruction file into a seat's home: no `CLAUDE.md` or `AGENTS.md` literal exists under `ai/services/fleet`, `ai/scripts/fleet` or `src/fleet` at `7d5af71` (control: the same grep finds the `mcp-config.json` literal). A Fleet seat started on the Brain, the Institution or the Skills repository therefore runs with no maintainer instructions at all. That includes `§critical_gates` and gate 10, which governs the Brain's own `ai/` config. Today a peer gets the rules only by running from an Engine checkout.

## The Architectural Reality

- **Homes.** `deriveHarnessLaunchSpec` points each seat's harness at its `instanceHome`: `CLAUDE_CONFIG_DIR` (`deriveHarnessLaunchSpec.mjs:50`) and `CODEX_HOME` (`:62`). The Codex home is `instanceHome`, or `instanceHome/codex-home` for `codex-desktop` (`prepareManagedAgentWorkspace.mjs:721`). Each harness reads a user-scope instruction file there in every session: Claude `CLAUDE.md`, Codex `AGENTS.md`. The loading facts and their sources are in neomjs/neo-agent-skills#100 (The Architectural Reality).
- **The seat's repository.** A registry row names one, `agent.metadata.repo.repoSlug` (`inspectFleetRepos.mjs:43`). The Skills source declares repositories by bare name (`neo`, `neo-agent-brain`, …), so `neomjs/<name>` maps to `<name>`. No other owner is declared there.
- **The composition.** The Brain already depends on `neo-agent-skills` (`package.json:108`, `^0.1.19`). From 0.1.23, `generate({audience: 'maintainer', repos})` is importable and refuses an undeclared repository, a list-number collision and output over 24,576 B.
- **Convergence.** `convergeTextArtifact` (`prepareManagedAgentWorkspace.mjs:2120`) creates a missing file, matches an unchanged one and throws `FLEET_WORKSPACE_DIVERGENT` otherwise. Generated instructions change with every Skills release, so through that path alone each release would refuse the next start. `convergeTransportReceipt` (`:1691`) already keys an owned write on a receipt of Fleet's last projection.
- **Claude loads both scopes.** A user-scope `CLAUDE.md` and a project `.claude/CLAUDE.md` load together, under no shared cap. An Engine seat given a home file today would load about 24 KB of the same rules twice per turn.
- **Codex reads its home file whole, beside the project budget.** `load_from_codex_home` reads `AGENTS.override.md` or `AGENTS.md` from `CODEX_HOME` with no size limit. `load_project_instructions` spends `project_doc_max_bytes` (32 KiB by default) on the project's files alone and truncates the file that crosses it (openai/codex `codex-rs/codex-home/src/instructions/mod.rs` and `codex-rs/core/src/agents_md.rs`, read at 50d77959b). So an Engine Codex seat given a home file would load both maintainer files in full; the budget deduplicates nothing.

## The Fix

Two parts, split so the preparer keeps one file-safety path:

- **The projection** is a sibling module, `ai/services/fleet/projectSeatInstructions.mjs`, and it writes nothing. It decides the target and the content (items 1, 2 and 4 below) and is pure enough to test without a filesystem.
- **The write** stays in `prepareManagedAgentWorkspace.mjs`, called once from `prepareHarnessArtifacts`. `assertNoSymlinkSegments`, `publishTextAtomically` and the receipt helpers are private there, and a second module writing into the home would copy them.

1. **The target follows the harness.** `claude-code` writes `<instanceHome>/CLAUDE.md`; `codex` and `codex-desktop` write `AGENTS.md` in the Codex home. Other harness types report `not-applicable` until their user-scope slot is witnessed.
2. **The content is the Skills composition.** `generate({audience: 'maintainer', repos: [name]})` for a `neomjs/<name>` seat. Another owner or an undeclared name reports `not-applicable` and starts normally; an outside institution's audience is a leaf of neomjs/neo-agent-institution#351.
3. **Convergence is keyed on a receipt.** `convergeTextArtifact` gains a `receiptPath` option: create when absent, replace when the file still hashes to Fleet's last write (a new Skills release), and refuse the start when a person edited it. The receipt, `.neo-fleet-seat-instructions.json` (`{version, artifact, sha256}`), sits beside the transport receipt, which keeps its own authenticated shape.
4. **A repository's own instruction file wins.** When the checkout carries a file the seat's harness reads as project instructions (Claude: `CLAUDE.md`, `.claude/CLAUDE.md`; Codex: `AGENTS.override.md`, `AGENTS.md`), the home copy is skipped and logged `repository-supplied`. Once the Engine deletes its tracked copies (neomjs/neo-agent-skills#100, Out of Scope), the next start writes the home file, with no flag.
5. **The repository travels beside the checkout.** The logical plan stays `{id, harnessType}`; `repoSlug` is a host fact about the checkout, read from the registry row and passed to the apply step next to `targetRepoRoot`.
6. **A skip is logged, not an artifact.** The artifact list stays the files Fleet owns in the workspace; `not-applicable` and `repository-supplied` go to the preparer's log with their reason.

*Prescription checked (intake stage 2): `ai/services/fleet/prepareManagedAgentWorkspace.mjs` owns the write, so the sibling module holds only the projection.*

### Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `<instanceHome>/CLAUDE.md` for `claude-code` | `deriveHarnessLaunchSpec` `CLAUDE_CONFIG_DIR` | the seat repository's maintainer composition | `repository-supplied` when the checkout carries a Claude file the harness can read (a symlink to one counts; a directory, a link to nothing or an unreadable file does not, and is named in `ignored`); `not-applicable` for an undeclared repository. When the seat stops taking the file, Fleet retires the copy it wrote; an edited copy refuses the start, and one Fleet never wrote stays and is reported | module JSDoc | a Brain-repository fixture seat gets the file; an Engine fixture seat is skipped |
| `AGENTS.md` in the Codex home for `codex` / `codex-desktop` | `deriveHarnessLaunchSpec` `CODEX_HOME`; the home root in `prepareCodexArtifacts` | the same composition | `repository-supplied` when the checkout carries a readable `AGENTS.override.md` or `AGENTS.md` (same entry rules); `not-applicable` for an undeclared repository; the same retirement on a transition | module JSDoc | a fixture arm per Codex type; an `AGENTS.md` checkout is skipped |
| the instructions receipt | the `convergeTransportReceipt` precedent | records Fleet's last write | a hand-edit refuses the start with `FLEET_WORKSPACE_DIVERGENT`; retirement removes the receipt with its file | module JSDoc | a release-update arm, a hand-edit arm, and a retirement arm per harness |
| the decision, `seatInstructions` | the preparation result; `startAgentProvisioned`'s returned status | `{state, reason, ignored?, homeFile?}` on every prepared start | a harness with no slot reports `not-applicable` with no path | JSDoc on both | a real composed start reports `repository-supplied` and `not-applicable` with no logger injected |

**Accretion disposition.** One new module that writes nothing; the preparer gains one call, a receipt option on an existing function and two small receipt helpers. The Codex Desktop home rule becomes one local helper shared by the Codex adapter and the instructions step instead of a second inline copy. Consumers' turn-loaded bytes fall once the Engine removes its tracked copies. Sunset: the `repository-supplied` branch retires when no declared repository tracks an instruction file its harness reads.

## Decision Record impact

`none`. No `ai/config*` leaf is added; every input is already on the registry row or the launch contract.

## Acceptance Criteria

- [ ] AC-1: A `claude-code` seat whose repository tracks no Claude instruction file gets `<instanceHome>/CLAUDE.md` holding the maintainer composition for its repository. A seat whose repository tracks one gets no home file, and the start reports `repository-supplied`.
- [ ] AC-2: A `codex` or `codex-desktop` seat gets `AGENTS.md` in its Codex home holding the same composition. A seat whose checkout carries `AGENTS.override.md` or `AGENTS.md` gets none, and the start reports `repository-supplied`.
- [ ] AC-3: A seat on an undeclared repository, or on a harness type without a witnessed slot, starts normally with no home file, and the start reports `not-applicable`.
- [ ] AC-4: A new Skills release replaces a file Fleet wrote; a hand-edited file refuses the start with `FLEET_WORKSPACE_DIVERGENT`, naming the path.
- [ ] AC-5 (post-merge, residual owner #571, the seat-launch outcome; acceptance trail neomjs/neo-agent-institution#12): a Fleet-started Claude seat and a Fleet-started Codex seat on the Brain repository each show the home file's rules in effect in their first session, with no repository instruction file present.

## Out of Scope

- **Skills in the home.** @neo-gpt-emmy's defect-note of 2026-09-30 16:15Z; #571 and neomjs/neo-agent-institution#12.
- **Multi-repository seats.** The registry holds one `repoSlug`. When attaching several lands, the caller passes the set, which `generate` already composes.
- **Removing the Engine's tracked copies.** neomjs/neo-agent-skills#100, Out of Scope.
- **`claude-desktop`, `kimi-code` and `opencode`**, until each harness's user-scope slot is witnessed.
- **DevIndex's hand-maintained `AGENTS.md`.**

## Avoided Traps

- **Writing through `convergeTextArtifact` alone.** Each Skills release would refuse the next start.
- **Writing the Claude home file into Engine seats now.** About 24 KB of the same rules would load twice per turn.
- **Treating Codex's byte budget as deduplication.** The home file is read whole, outside `project_doc_max_bytes`, and the project file that crosses the budget is truncated, not skipped; both copies would load.
- **A second generator, or sections copied into Fleet.** The Skills package is the source of record (the responsibility split on neomjs/neo-agent-skills#100).
- **Failing a start for an undeclared repository.** That is a tenant seat, not an error.

## Related

- neomjs/neo-agent-skills#100 — the composition export this calls (0.1.23).
- #571 — the seat folder layout; the module follows `instanceHome` wherever that lands.
- #642 — Fleet seeds the repository's Codex context into the same home.
- neomjs/neo-agent-institution#12 — FM onboarding; owns the installed witness.
- neomjs/neo-agent-institution#351 — the outside operator's first run and its own audience.

## Handoff Retrieval Hints

- `query_raw_memories`: `"Fleet seat harness home instructions CLAUDE.md AGENTS.md generate agents-md receipt"`
- Reproduce the gap: `gh api 'repos/neomjs/neo-agent-brain/contents/AGENTS.md?ref=dev'` returns 404.

Origin Session ID: 8c224931-7b3d-4cb5-a43d-86f1735f3636





## Timeline

- 2026-09-30T17:14:48Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-30T17:14:49Z @neo-opus-grace added the `enhancement` label
- 2026-09-30T17:14:49Z @neo-opus-grace added the `ai` label
- 2026-09-30T17:14:49Z @neo-opus-grace added the `agent-os` label
- 2026-09-30T17:15:31Z @neo-opus-grace cross-referenced by #100
- 2026-09-30T17:16:25Z @neo-opus-grace cross-referenced by PR #127
- 2026-09-30T18:13:01Z @neo-opus-grace cross-referenced by #645
- 2026-09-30T18:13:22Z @neo-opus-grace cross-referenced by #19335
- 2026-09-30T18:19:16Z @neo-opus-grace referenced in commit `38e6f16` - "feat(fleet): project a seat's maintainer instructions into its harness home (#644)"
- 2026-09-30T18:19:16Z @neo-opus-grace referenced in commit `5f534a6` - "feat(fleet): converge a seat's instructions in its harness home against a receipt of Fleet's last write (#644)"
- 2026-09-30T18:24:05Z @neo-opus-grace referenced in commit `bd13b26` - "fix(fleet): a checkout's AGENTS.md supplies a Codex seat's instructions, since Codex reads its home file beside the project budget (#644)"
- 2026-09-30T19:23:06Z @neo-gpt-emmy cross-referenced by #648
- 2026-09-30T19:35:03Z @neo-gpt cross-referenced by PR #647
- 2026-09-30T19:40:20Z @neo-opus-grace cross-referenced by PR #649
- 2026-09-30T20:23:36Z @neo-opus-grace cross-referenced by PR #131
- 2026-09-30T21:10:48Z @neo-opus-grace referenced in commit `304c62d` - "feat(fleet): project a seat's maintainer instructions into its harness home (#644)"
- 2026-09-30T21:10:48Z @neo-opus-grace referenced in commit `7385354` - "feat(fleet): converge a seat's instructions in its harness home against a receipt of Fleet's last write (#644)"
- 2026-09-30T21:10:48Z @neo-opus-grace referenced in commit `dd280f5` - "fix(fleet): a checkout's AGENTS.md supplies a Codex seat's instructions, since Codex reads its home file beside the project budget (#644)"
- 2026-09-30T21:10:48Z @neo-opus-grace referenced in commit `7532838` - "feat(fleet): Fleet resolves neo-agent-skills ^0.1.23, the first release that exports agents-md (#644)

projectSeatInstructions imports neo-agent-skills/agents-md, which 0.1.23 (Skills #127) is the first to export; the floor moves from ^0.1.19 so no install can resolve a release without it."
- 2026-09-30T21:10:51Z @neo-opus-grace cross-referenced by PR #654
- 2026-09-30T22:07:04Z @tobiu referenced in commit `bb9350d` - "feat(fleet): a seat's instructions converge both ways, count only a readable checkout file, and report the decision on the start (#644)

When the seat stops taking its home file (the checkout now supplies one, or the repository is undeclared), the file Fleet wrote is retired with its receipt; a file edited since refuses the start, and one Fleet never wrote stays and is reported. A checkout entry supplies instructions only when it resolves to a readable file, a symlink to one included; a directory, a link to nothing or an unreadable file falls back to the composition and is named. The preparation result carries the decision as seatInstructions, and startAgentProvisioned returns it on the status its caller already receives."
- 2026-09-30T22:23:31Z @tobiu referenced in commit `5653203` - "Merge pull request #654 from neomjs/grace/644-seat-instructions

feat(fleet): a Fleet seat starts with its repository's maintainer instructions in its harness home (#644)"
- 2026-09-30T22:23:31Z @tobiu closed this issue
- 2026-09-30T22:24:51Z @neo-opus-grace cross-referenced by #571

