---
id: 4
title: Harness packaging accepts separate product and Brain roots
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - architecture
  - build
  - testing
assignees:
  - neo-fable-clio
createdAt: '2026-08-27T09:01:28Z'
updatedAt: '2026-09-04T19:43:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/4'
author: tobiu
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1 The Fleet Manager arrives with its own test suite'
blocking: []
closedAt: '2026-09-04T19:43:29Z'
---
# Harness packaging accepts separate product and Brain roots

## Context

The Electron harness now lives in Institution and resolves its live Brain through an explicit absolute checkout root. Its packaging stage still assumes the pre-split monorepo.

Live latest-open sweep: checked immediately before creation at 2026-08-27T09:01:26.816Z; no equivalent Institution ticket existed.

## The Problem

`harness/pack.mjs` still copies `ai/**` and `buildScripts/util/sanitizer.mjs` from the Institution root and optionally reads a co-located `package.brain.json`. None of those paths exists here, so `npm --prefix harness run dist` cannot assemble the post-split product.

The test receiver intentionally did not patch individual paths: that would create a half-split artifact while keeping implicit custody.

## Reviewer-discovered launcher-ordering follow-up

`harness/main.mjs` currently resolves `NEO_AGENTOS_RUNTIME_ROOT` and loads Fleet contracts before `resolveBrainMode()` can honor checkout mode with `NEO_HARNESS_BRAIN=0`. The path is unreachable today because Institution exposes no harness launcher, but this ticket must make the root/contracts lazy or Brain-mode-gated before wiring one.

## The Architectural Reality

- Institution owns the application, Electron shell, product assets, and packaging entrypoint.
- Engine code comes from the pinned `neo.mjs` dependency.
- Brain executables and their dependency authority come from the explicit `NEO_AGENTOS_RUNTIME_ROOT`.
- Packaged mode may assemble those inputs, but checkout discovery and cwd/sibling fallbacks remain forbidden.

Decision Record impact: aligned with the completed repository split.

## The Fix

Refactor the pack stage around explicit product, Engine-package, and Brain roots. Derive the staged runtime graph and manifest from those owners, retain the instance-overlay security stop-line, and fail before mutation when any required root or declaration is missing.

## Acceptance Criteria

- [ ] Packaging succeeds from a clean Institution checkout with an explicit absolute Brain root.
- [ ] No local `ai/**`, `package.brain.json`, or Neo-source mirror is required.
- [ ] Missing/relative Brain authority fails before staging or dependency installation.
- [ ] The staged manifest derives dependencies from the actual product and Brain package authorities.
- [ ] Instance overlays remain excluded and the packaged-runtime import closure is verified.
- [ ] Unit coverage proves the split-root assembly rather than the former monorepo shape.
- [ ] A checkout launcher can run with Brain mode disabled without requiring a Brain root or loading Brain contracts. `[L3-deferred — operator handoff needed]` — unit arms + the static boot path land with PR #109; the live receipt is owned by #17 (see the Contract Ledger below).

## Out of Scope

- Code signing, notarization, or publishing.
- Changing Institution product identity.
- Copying Brain source into the repository.

## Related

Related: #1

Origin Session ID: d39e8182-295f-418a-82cd-a96be9c08e4f

Retrieval Hint: "Institution harness pack explicit product Engine Brain roots"

Authored by Emmy (GPT-5.6 Sol Ultra, Codex). Session d39e8182-295f-418a-82cd-a96be9c08e4f.


## Contract Ledger (claimer-authored, intake-derived — corrected per PR #109's review)

*Appended 2026-09-04 by the claimer under the foreign-ticket restatement rule; the body above is the author's.*

| Surface | Source of authority | Proposed behavior | Fallback / failure | Docs | Evidence |
|---|---|---|---|---|---|
| Brain root — `NEO_AGENTOS_RUNTIME_ROOT` | `harness/brain.mjs#resolveAgentOsRuntimeRoot` (absolute-only, existing) | the pack stage's Brain authority: its `ai/` and `src/` trees are staged, its `package.json` is the runtime-dependency tier; proven (markers) before `rm`, the theme build, or any install | missing / relative → the contract's TypeError; a root without `package.json`, `ai/` or `src/` → named, nothing mutated | `harness/README.md` § Packaging | `pack.spec.mjs` › *resolvePackRoots proves…*, › *stageOrganism fails before mutation…* |
| Product root — this checkout | the contentPolicy allowlist (renderer graph) + the product `package.json` | stages the allowlist-derived graph; pins its own trees' imports; owns the Engine pin | missing `apps/agentos` / `package.json` → named, nothing mutated | same | same arms |
| Engine package — `node_modules/neo.mjs` | the product's `neo.mjs` pin | supplies the theme builder (`buildScripts/build/themes.mjs`) at pack time and the Engine the staged install resolves; `neo.mjs` is the one named owner exception (`OWNER_EXCEPTIONS`) | missing package / builder → named, nothing mutated | same | › *resolvePackRoots…* (engine marker), › *…the Engine is the product's by name* |
| Staged manifest (`.stage/organism/package.json`) | per-owner provenance: each staged tree's bare imports pinned by the owner that stages it, both tiers of each manifest | a name both trees import must carry the same declaration in both owners; an import its owner never declared is a hard error; supplemental names belong to an owner | a disagreement fails the pack loud (no silent precedence) | same | › *…keeps owner authority at a collision…* (red control: `chalk` ^6 vs ^5.6.2) |
| Import closure of the stage | `pack.mjs#assertImportClosure` | after install + fresh configs: every pinned package present under the stage's `node_modules`, every relative import of a staged file resolves inside the stage | a missing package or a dangling relative import fails the pack, naming it | same | › *assertImportClosure…* + the real stage run |
| `organism-build-info.json` | `pack.mjs#describeOwners` | role, name, version, the Engine pin, the Brain revision — never a build-host path (the file rides `extraResources` verbatim) | a missing revision is `null`, never a guess | same | › *describeOwners ships…* |
| `NEO_HARNESS_BRAIN=0` on a checkout, no root | `brain.mjs#resolveLauncherRuntimeRoot` | no root resolved, no Fleet contract loaded, bearer/transport story `null`, the fleet IPC route rejects with the named reason (`createAbsentFleetCapability`) | a relative root stays an error; an untrusted sender is refused first | `main.mjs` header | › *resolveLauncherRuntimeRoot…*, `fleetCapability.spec.mjs` › *the absent capability rejects…* |

**AC-7 disposition:** `[L3-deferred — operator handoff needed]` — the unit arms and the static boot path are the PR's evidence (L2); the live receipt (a checkout launch with the Brain leg off and no root boots the UI alone and logs `HARNESS_UI_FLEET none`) needs an Electron install this seat's machine does not carry. Owner of that exact receipt: #17 (its Sequencing section names it).


## Timeline

- 2026-08-27T09:01:29Z @tobiu added the `bug` label
- 2026-08-27T09:01:29Z @tobiu added the `agent-os` label
- 2026-08-27T09:01:30Z @tobiu added the `ai` label
- 2026-08-27T09:01:30Z @tobiu added the `architecture` label
- 2026-08-27T09:01:31Z @tobiu added the `build` label
- 2026-08-27T09:01:31Z @tobiu added the `testing` label
- 2026-08-27T09:01:40Z @neo-gpt-emmy marked this issue as being blocked by #1
- 2026-08-27T09:03:08Z @tobiu cross-referenced by #1
- 2026-08-27T09:04:00Z @tobiu cross-referenced by PR #5
- 2026-08-27T09:09:56Z @neo-gpt-emmy cross-referenced by PR #6
- 2026-08-27T11:12:44Z @neo-gpt cross-referenced by #25
- 2026-08-27T11:25:25Z @tobiu cross-referenced by PR #26
- 2026-08-28T15:41:30Z @neo-fable-clio cross-referenced by PR #35
- 2026-09-04T18:10:11Z @neo-fable-clio assigned to @neo-fable-clio
### @neo-fable-clio - 2026-09-04T18:10:27Z

## Intake — `valid-as-written`, claimed

**Premise, falsified against both checkouts (2026-09-04, `dev@0b0b33a`):** the Institution root carries no `ai/`, no `package.brain.json`, no `buildScripts/util/sanitizer.mjs`; `harness/pack.mjs` still stages `BRAIN_TREES = ['ai']` (:40), `BRAIN_FILES = ['buildScripts/util/sanitizer.mjs']` (:44), reads `package.brain.json` (:395) and runs `ai/scripts/setup/initServerConfigs.mjs` inside the stage (:413) — every one of them from `repoRoot`. The Brain checkout has all four (its `ai/`, the setup script, the sanitizer, the `examples` and `temporal-summary` excludes); only `ai/scripts/diagnostics/genesisProbe.mjs` (`TREE_EXCLUDES` :55) exists nowhere any more — a dead exclude, harmless, noted. The launcher half holds too: `harness/main.mjs:70` calls `resolveAgentOsRuntimeRoot()` at module top level, which THROWS without an absolute `NEO_AGENTOS_RUNTIME_ROOT` (`brain.mjs:54-65`), and `:81` loads the Fleet contracts before `brainMode` (:80) gates anything — `NEO_HARNESS_BRAIN=0` cannot reach a checkout boot today.

**Ticket age / successor risk:** created 2026-08-27, untouched since; no commit on `harness/pack.mjs`, `brain.mjs`, `main.mjs` or `pack.spec.mjs` since that day; no newer ticket or PR on the surface (live latest-open + KB `type: ticket` + Memory Core sweep: the Electron-shell lineage #13012 / #13033 / #13377 only). No parent epic, so no epic-review gate. ADR: aligned with the split (the shell ADR's E6 row stays the binding).

**Surfaces the fix touches** (the ticket carries no ledger; recorded here, claimer-authored):

| Surface | Authority | Proposed | Fallback |
|---|---|---|---|
| `NEO_AGENTOS_RUNTIME_ROOT` | `brain.mjs#resolveAgentOsRuntimeRoot` (absolute-only, existing) | reused as the pack stage's Brain-root authority; the Brain's `package.json` replaces `package.brain.json` as the runtime-dependency tier | missing / relative → throw before `rm`, before the theme build, before `npm install` |
| product root | the Institution checkout (`apps/agentos`, `resources`, `dist/development/css` per the allowlist) | unchanged owner; validated to exist | missing marker → throw before mutation |
| Engine | `node_modules/neo.mjs` (the pinned dependency) | the staged manifest pins it from the product `package.json`; on a name collision the product declaration outranks the Brain's | undeclared import → the existing hard error |
| `organism-build-info.json` | pack output | gains the resolved roots + Brain name/version for provenance | — |
| `NEO_HARNESS_BRAIN=0` on a checkout | `resolveBrainMode` (existing) | root resolution + `loadFleetRuntimeContracts` become Brain-mode-gated; the shell's fleet capability renders its honest absent form | — |

Branch `agent/4-harness-split-roots` off `dev`; unit coverage lands in `pack.spec.mjs` (the monorepo-shaped arms restated), and `harness/README.md`'s packaging section names the Brain-root requirement.

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 46962d8b-08f3-49a3-8049-d74e2052af37

- 2026-09-04T18:26:56Z @neo-fable-clio cross-referenced by PR #109
- 2026-09-04T18:58:55Z @neo-fable-clio cross-referenced by #17
- 2026-09-04T19:05:22Z @neo-fable-clio referenced in commit `2feebb4` - "fix(harness): the manifest keeps each import's owner, the build info ships no path, the Brain's src rides with ai, the closure covers relative imports (#4)

Review round 1 (Euclid): the manifest flattened both owners with product-last precedence — a Brain import of chalk took the product's ^6 over the Brain's ^5.6.2 — and organism-build-info.json carried absolute checkout paths into extraResources. Now each staged tree's bare imports are pinned by the owner that stages it, a name both trees import must carry the same declaration in both owners (a disagreement fails the pack), neo.mjs is the one named owner exception, supplemental names belong to an owner, and describeOwners records role, name, version, the Engine pin and the Brain revision only.

Measured on the staged tree while answering: ai/ imports the Brain's own src/ (composition, evolution) relatively and the stage never copied it — the product's src/ sat at that path; three staged modules dangled and a fourth imported the excluded temporal-summary daemon. BRAIN_TREES gains src (beside the product's src: distinct subtrees, a same-path collision fails the copy), the aggregation script joins the exclude, the sanitizer copy is dropped (ai/ imports it bare from the Engine package), and assertImportClosure resolves every relative import of a staged module on the COMPLETE stage, after the fresh configs.

Arms: the collision red control (chalk ^6 vs ^5.6.2), a Brain-only import pinned by the Brain, the neo.mjs exception, strict per-owner missing, supplemental per owner, describeOwners over fake absolute roots (no path survives), the closure's dangling-import case. Harness specs 76, unit 767."
- 2026-09-04T19:05:22Z @neo-fable-clio referenced in commit `8fd6a3d` - "fix(harness): the stage fills the overlay slots the setup script leaves empty, and the closure walks a shared tree once (#4)

The real stage at round 2 passed everything up to the new closure check and stopped at src/evolution/RemDigestion.mjs → ./config.mjs: an instance-overlay slot of the Brain's own src/ (template + gitignored config) that the copy filter rightly leaves out and that the Brain's initServerConfigs never fills — it knows ai/config.mjs and ai/mcp/server/*/config.mjs only. materializeOverlaySlots fills any such slot template-verbatim from the same derived predicate, after the setup script and before the closure check; a slot the script filled is left alone. The closure walked src twice (both owners name it) and reported the finding twice — walked once now.

Real stage at this head: 5 trees + 2 files, 22 pins, @electron/rebuild on 43.1.0, one slot materialized, closure green, owners without a path."
- 2026-09-04T19:23:05Z @neo-fable-clio referenced in commit `6d3dac9` - "fix(harness): each owner is scanned from the files it copied — the merged stage cannot show provenance (#4)

Round 2 (Euclid): buildOrganismManifest kept provenance but stageOrganism scanned the merged stage, and both owners land a src/ there — every bare import in either owner's src/ was credited to both, so a Brain-only import declared by the Brain alone read as missing for the product (and the reverse). copyTree/copyFile now return the stage-relative files they placed; stageOwners is the copy-and-scan boundary — each owner scanned from ITS files through collectBarePackages over an explicit set (the directory walker is gone); the overlay stop-line runs there. The integration arm over fake roots runs that boundary with the merged scan as the red control and a same-file collision as the second; harness specs 77, unit 768."
- 2026-09-04T19:43:30Z @tobiu closed this issue
- 2026-09-04T19:43:30Z @tobiu referenced in commit `99b8911` - "Merge pull request #109 from neomjs/agent/4-harness-split-roots

fix(harness): the pack stage assembles from explicit product, Engine and Brain roots, and the launcher gates the Brain root on the Brain leg (#4)"

