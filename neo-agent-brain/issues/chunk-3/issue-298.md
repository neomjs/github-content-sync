---
id: 298
title: Two invariants crossed the split without the specs that witness them
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - regression
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-02T02:41:31Z'
updatedAt: '2026-09-02T12:00:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/298'
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
closedAt: '2026-09-02T12:00:45Z'
---
# Two invariants crossed the split without the specs that witness them

## Context

Engine commit `c623b2f63c` (the AgentOS extraction, `neomjs/neo#17791`) deleted 804 unit specs, 30 of them under `test/playwright/unit/ai/buildScripts/`. `neomjs/neo#17922` is the Engine-side census of that loss, and its [02:30Z disposition ruling](https://github.com/neomjs/neo/issues/17922) resolved the residual set **per arm** rather than per file: 25 arms restore in the Engine, 2 retire with the parity lane that left, and **2 assert properties whose subjects now live in this repository**.

Those two arms cannot be restored in the Engine — their subjects are here. They also cannot be dropped: both properties are live and correct at `dev@94c1df9f0b`, and neither has a witness in this repo. This ticket receives them.

The general shape is worth stating once, because it is the reason both were invisible: **the severance was censused for what moved, never for what stayed** — and these two are the mirror case, a spec that stayed while the thing it describes left. The source repo reads green because the spec is gone; this repo reads green because it never received one. Both halves report success while the coverage is dead in the middle.

## The Problem

### Arm 1 — the logical-identity guard before every broad release stage

From the deleted `release/PublishReleaseNoteOrphan.spec.mjs`, arm #6: *every broad-staging (`git add .`) release commit is preceded by the logical-identity guard.* Its own header explains the shape it was written in:

> The population is pinned per file — a count over one file went stale the day the split landed, and a mere total would let both stages migrate into one unguarded file and still pass.

The arm anticipated this exact split and pinned per file to survive it. `c623b2f63c` then deleted the whole spec.

**The invariant holds here today, and nothing witnesses it.** Measured at `dev@94c1df9f0b`:

| fact | evidence |
|---|---|
| the guard call | `ai/scripts/lifecycle/postReleaseSync.mjs:224` — `assertNoArchiveLogicalIdentityCollisions('commit archived tickets', root)` |
| the broad stage it protects | `:225` — `runCommand('git add .', 'Failed to stage archive changes')` |
| the entrypoint is live | `package.json:78` — `"ai:post-release-sync": "node ./ai/scripts/lifecycle/postReleaseSync.mjs"`, and ADR 0040 §2.3 names it as the Brain half of the two-command release seam |
| coverage | `git grep -ln postReleaseSync -- 'test/**'` → **no hits, anywhere in the repo** |

**The near-miss is what makes this dangerous.** `test/playwright/unit/ai/scripts/lifecycle/` holds 21 specs, one of which is `postReleasePreflight.spec.mjs` — a *different* file (`postReleasePreflight.mjs`), guarding a different claim (`assertAdmissibleStartingState`: what may be pending *before* the broad stage). Any sweep that greps the directory listing for `postRelease` concludes the release path is covered. It is half-covered, and the uncovered half is the one that stages.

Three properties make a source-ordering spec the only available instrument here:

1. Both release commits run `--no-verify` by design, so husky and `lint-staged` are structurally blind to them.
2. Proving it behaviourally would require cutting a real release.
3. This stage sits **deliberately after** a `catch` that continues when `runFullSync()` has thrown — see the source comment at `:221`. It therefore fires exactly when the corpus-integrity verdict has already refused the corpus, which is the highest-risk moment to stage broadly.

To be explicit: **this is not a live defect.** Both call sites are correctly ordered at HEAD. What is missing is the ability to notice if they stop being.

### Arm 2 — the corpus re-chunk, ordered before the emitter's own derive

From the deleted `docs/RebuildContentIndexesAndSeo.spec.mjs`, arm #3. That arm had two halves and they part company at the split:

- **Half A** — the Engine module imports nothing from `ai/**`. Stays Engine-side under `neomjs/neo#17922`; it is a module-level witness for **ADR 0040 §2.3**, *"The Engine never imports from the extracted repository."*
- **Half B** — the re-chunk pass did not simply vanish: the emitter carries it, ordered before its own derive call, so every reader projects an already-canonical corpus. **That emitter is now `ai/services/github-workflow/SyncService.mjs`, in this repository.**

The property holds at `dev@94c1df9f0b`: `await reconcileActiveChunks(...)` at `:312` / `:313` / `:314` (pulls, issues, discussions), `await this.rebuildContentIndexesAndSeo()` at `:320`.

**No spec in this repo witnesses it.** Checked all three candidates rather than inferring from a directory listing:

| candidate | what it actually asserts | covers arm 2? |
|---|---|---|
| `SyncService.Stage2.spec.mjs:263` | ordering `metadata-save → derive → permission-check`; **zero** references to `reconcileActiveChunks` | no |
| `shared/reconcileActiveChunks.spec.mjs` | the helper's own ranking and dedup behaviour, in isolation | no — never reaches the call site |
| `ProcessSupervisorService.spec.mjs:118` | log-level classification; matches the string `[reconcileActiveChunks]` only | no |

## The Architectural Reality

- `ai/scripts/lifecycle/postReleaseSync.mjs` — Brain half of the release seam; `assertNoArchiveLogicalIdentityCollisions` is defined at `:73` and called at `:224`.
- `ai/services/github-workflow/SyncService.mjs` — the corpus emitter; `emitGeneratedContentAndDerive` owns the ordinal-100 re-chunk that moved here from the Engine's projection script.
- `test/playwright/unit/ai/scripts/lifecycle/` and `test/playwright/unit/ai/services/github-workflow/` — both destination directories exist with sibling specs and an established fixture idiom, so placement is a sibling lift, not a novel structural choice.
- **ADR 0040 §2.3 / §2.5** — the dependency direction that makes both subjects Brain-owned in the first place.

### The honest CI bound

`.github/workflows/brain-unit.yml` runs `npm run test-unit -- --list` (which collects) and then executes **four named smoke specs**. Specs added by this ticket will **not** execute in Brain CI. `#201` owns that reach defect and is not in scope here. Consequence for this ticket: **the acceptance evidence is a local targeted run, and a green `brain-unit` check is not evidence for any AC below.** Claiming otherwise would reproduce, one repo to the right, the exact failure `neomjs/neo#17922` is about — green surviving the loss of the coverage.

## The Fix

1. **New** `test/playwright/unit/ai/scripts/lifecycle/postReleaseSync.spec.mjs` — witness that the logical-identity guard precedes the broad stage, pinned **per call site** (the original's deliberate shape), not as a file-wide total.
2. **Extend** the `SyncService` unit coverage with the call-site ordering arm: the re-chunk pass runs, and it runs before `rebuildContentIndexesAndSeo()`.

Two prescriptions are about the invariant, not the line, and both deviate deliberately from a verbatim restoration:

- **Prefer a behavioural arm for #2 if the import seam allows it.** `SyncService.Stage2.spec.mjs` already stubs `SyncService.rebuildContentIndexesAndSeo` and records an `order` array; if `reconcileActiveChunks` can be intercepted at its import seam, the ordering is witnessed behaviourally, which strictly dominates a source-text `indexOf` comparison. Fall back to source-ordering only if the seam does not permit it, and say so in the spec.
- **Do not restore the original's `toHaveLength(3)`.** The 3 is one call per corpus facet (pulls / issues / discussions); pinning the total makes the arm fail the day a fourth facet is added while saying nothing about the property it exists to protect. Assert one re-chunk per facet, keyed by facet.

## Decision Record impact

`aligned-with ADR 0040` (§2.3 dependency direction, §2.5 two root authorities). This ticket adds witnesses for properties the ADR already establishes; it amends nothing.

## Acceptance Criteria

- [ ] A spec exists that fails when `assertNoArchiveLogicalIdentityCollisions` is removed from, or reordered after, the `git add .` at `postReleaseSync.mjs:225` — demonstrated by a seeded mutation, red first, in both directions.
- [ ] That spec's population is pinned **per broad-stage call site**, so a second broad stage added to the file without a guard fails it. A file-wide count does not satisfy this.
- [ ] A spec exists that fails when the re-chunk pass is removed from `SyncService`, and separately when it is reordered after `rebuildContentIndexesAndSeo()` — again seeded and red first, both mutations.
- [ ] That spec asserts one re-chunk per corpus facet keyed by facet, and does **not** pin a total call count.
- [ ] Evidence is a targeted local run naming the exact spec paths and pass counts. `brain-unit` green is explicitly **not** offered as evidence for any AC above (see *The honest CI bound*).
- [ ] `neomjs/neo#17922` is updated with the ticket number, closing its "2 arms transfer to the Brain" row.

## Out of Scope

- The 25 arms that restore in the Engine, and the 2 that retire with the parity lane — `neomjs/neo#17922` owns those.
- Half A of arm #3 (the Engine module's `ai/**` import boundary) — Engine-side, same parent.
- Making Brain unit CI select these specs — that is `#201`, and this ticket must not smuggle it in as an AC.
- Changing anything `postReleaseSync.mjs` or `SyncService.mjs` *does*. Both are correct at HEAD; restoring a witness is not the moment to fix the subject.
- The `check-spec-retirement` guard's surviving-subject narrowing (`neomjs/neo#17964`), and the broader question of how many *other* moved specs are dead in the middle of a severance. Flagged in the parent, unmeasured, deliberately not filed from inside this leaf.

## Avoided Traps

- **Reading `postReleasePreflight.spec.mjs` as coverage.** Adjacent name, different file, different claim. This is the trap that hid the gap.
- **Restoring the arms verbatim.** Both carry assertions calibrated to the pre-split tree; one pins a count that the split itself would have broken. A verbatim lift would import the staleness along with the property.
- **Treating a green `brain-unit` run as evidence.** It executes four smoke specs. Any AC phrased against that check's colour re-creates the exact blind spot this ticket descends from.
- **Trimming an assertion to make it pass.** If either property turns out not to hold at HEAD, that is a finding about the subject and gets its own ticket — not a weakened arm.
- **Deriving the specs from the Engine's deleted blob without re-measuring.** The blob is 5 days stale relative to this repo; every line number above was read at `dev@94c1df9f0b`, and an implementer should re-read rather than trust this body's numbers.

## Sweep attestations

- **Live latest-open sweep:** checked the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-02T02:38Z, created-descending; no equivalent found. Nearest: `#194` (make the retained Brain test suite real), `#201` (unit-workflow CI reach), `#292` (retire the twelve Engine-owned specs the split left here) — all adjacent, none covering a *received* witness.
- **Closed-issue sweep:** `state:closed` over spec/coverage/split terms; `#292` is the retire direction of this same severance, this is the receive direction. No decline to re-litigate.
- **A2A in-flight claim sweep:** `list_messages({status:'all', limit:30})` at 02:38Z — no `[lane-claim]` or `[lane-intent]` from any peer on Brain test coverage, `postReleaseSync`, or `SyncService` in the herd window.
- **Memory Core rationale sweep:** `query_raw_memories` keyed on the observed symptom (release-lifecycle coverage absent after the extraction) returned nothing on-point. Stated as *nothing surfaced*, not *nothing exists*.
- **Own-assignment sweep:** 23 open issues assigned to me in this repo, read for same-surface overlap; none touch either subject.
- **Structure gate:** sibling-lift, both destinations. `test/playwright/unit/ai/scripts/lifecycle/` (21 sibling specs) and `test/playwright/unit/ai/services/github-workflow/` (existing `SyncService.Stage2.spec.mjs`). No novel directory choice.

## Relationship to `#194`

Filed standalone rather than as a sub of `#194` deliberately. `#194` is about making the tests *already retained here* real — execution, ownership, and honest CI reach over the existing tree. This ticket is about two witnesses that **never arrived**, and its authority chain runs through the Engine census `neomjs/neo#17922`, not through a Brain domain slice. If @neo-gpt-emmy reads it as belonging under `#194`, adopting it as a sub is welcome and I will not contest the call.

## Related

- `neomjs/neo#17922` — parent census; its 02:30Z disposition ruling is the source of both arms
- `neomjs/neo#17791` / `c623b2f63c` — the split commit that deleted the specs
- `neomjs/neo#18052` / PR `neomjs/neo#18053` — the sibling restoration that surfaced this residual set
- `#194` / `#201` — retained-suite reality and CI reach (see above)
- `#292` — the retire direction of this same severance
- `#289` / `#291` — GitHub corpus emission custody, same emitter

## Handoff Retrieval Hints

- `Retrieval Hint: "a moved spec whose subject stayed behind — both repos green, coverage dead in the middle"`
- `Retrieval Hint: Engine commit range c623b2f63c^..c623b2f63c, files test/playwright/unit/ai/buildScripts/release/PublishReleaseNoteOrphan.spec.mjs and docs/RebuildContentIndexesAndSeo.spec.mjs`
- Read the deleted arms directly: `git show c623b2f63c~1:test/playwright/unit/ai/buildScripts/release/PublishReleaseNoteOrphan.spec.mjs` in an Engine checkout.

Origin Session ID: 858be1c8-5348-4325-9aa7-8e95766f017e

Authored by @neo-opus-grace (Anthropic Claude Opus 5, Claude Code).

🖖 Grace

## Timeline

- 2026-09-02T02:41:32Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-02T02:41:33Z @neo-opus-grace added the `bug` label
- 2026-09-02T02:41:33Z @neo-opus-grace added the `ai` label
- 2026-09-02T02:41:33Z @neo-opus-grace added the `testing` label
- 2026-09-02T02:41:33Z @neo-opus-grace added the `regression` label
- 2026-09-02T02:41:33Z @neo-opus-grace added the `agent-os` label
- 2026-09-02T02:42:06Z @neo-opus-grace cross-referenced by #17922
- 2026-09-02T08:58:36Z @neo-opus-grace cross-referenced by PR #299
- 2026-09-02T12:00:45Z @tobiu referenced in commit `80c551e` - "Merge pull request #299 from neomjs/grace/298-received-witnesses

test(lifecycle): receive the two witnesses the split left in the middle (#298)"
- 2026-09-02T12:00:45Z @tobiu closed this issue

