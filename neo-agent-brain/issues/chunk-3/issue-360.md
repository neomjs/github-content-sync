---
id: 360
title: 'The tier gate proves a node-gyp artifact, so a shipped prebuild reads as a broken install'
state: CLOSED
labels:
  - bug
  - ai
  - testing
  - build
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T18:42:21Z'
updatedAt: '2026-09-15T20:11:52Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/360'
author: neo-opus-vega
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
closedAt: '2026-09-15T20:11:52Z'
---
# The tier gate proves a node-gyp artifact, so a shipped prebuild reads as a broken install

## Context

`#358` put "why the tier went partial" in **Out of Scope**, pending a PR where it would be attributable. It is attributable now, and the answer inverts what `#358` records.

`#357` is red on step 7 of its `unit` job (`Verify the received unit collection`):

```
[playwright.config.unit] CI requires the complete Brain tier (better-sqlite3,
chromadb, @chroma-core/default-embed) but it is absent or partial
```

**The tier is not partial. `better-sqlite3@13` ships its binary and the gate looks for it in v12's place.**

⛔ **`#358`'s "Avoided Traps" note is now wrong and this ticket supersedes it.** It says `better-sqlite3@13.0.3` "is falsified — it installs and runs standalone". That measurement was real but answered the wrong question: whether the *module* loads, never whether the *artifact the gate probes* exists. An instrument that cannot produce the failing result is not a falsifier.

## The Problem

`hasBrainTier` requires `build/Release/better_sqlite3.node` inside the `better-sqlite3` package directory. The two versions produce that file under opposite conditions:

| | `12.11.1` (on `dev`) | `13.0.3` (on `#357`) |
|---|---|---|
| `install` script | `prebuild-install \|\| node-gyp rebuild --release` | **none** |
| dependencies | `bindings`, `prebuild-install` | `node-addon-api` |
| `.node` files in the npm tarball | **0** | **8** (`prebuilds/{darwin,linux,linuxmusl,win32}-{x64,arm64}.node`) |
| where the binary lands | downloaded/compiled → `build/Release/` | already present → `prebuilds/<target>.node` |
| `build/Release/better_sqlite3.node` | created | **never created** |

Measured from the published tarballs (`npm pack` + `tar tzf`), so it is platform-independent metadata, not a local-host result.

v13's `binding.gyp` says so in upstream's own words:

```
# npm's implicit node-gyp rebuild should do nothing when the package
# contains a prebuild for the host. Explicit build scripts override this.
'prebuild_exists%': '<!(node lib/binding.js)',
```

With `prebuild_exists==1` both targets become `type: none`. So the workflow's **`npm rebuild better-sqlite3` is correct and correctly builds nothing** — it printed `rebuilt dependencies successfully` in 4.2s. Nothing in the install is broken.

**The corroborating asymmetry:** `prebuild-install@7.1.3` is in `dev`'s lockfile and **absent** from `#357`'s. That is the whole delta — v13 stopped needing a download step. The two other tier members are identical across both locks (`chromadb@3.5.0`, `@chroma-core/default-embed@0.1.9`), so neither is implicated.

**Not the discriminator, though it looks like one:** step 5 emits `npm warn rebuild 6 packages have install scripts not yet covered by allowScripts` — **in the green `dev` run too**, byte-identical. There is no `allowScripts` allowlist in this repository (no `.npmrc`, no `package.json` key). Anyone re-diagnosing this will meet that warning first; it is noise.

## The Architectural Reality

The gate's own `@summary` in `test/playwright/playwright.config.unit.mjs` argues for exactly this probe:

> *"resolution answers 'is there an entrypoint', never 'did the native build produce its artifact', so it would report armed for exactly the broken install this probe exists to catch"*

That reasoning is still right. What went stale is its unstated premise — **that there is a native build at all.** v13 removed the build from the default path, so a probe keyed to the build's output now reports a broken install for a healthy one. A husk check that hardcodes one loader's layout inherits that loader's version.

And `better-sqlite3@13` publishes the authority the gate should be asking: `lib/binding.js` exports `getPrebuildPath()`, and `getBinding()` searches `prebuilds/<linuxmusl|platform>-<arch>.node` → `build/Debug/…` → `build/Release/…`. The gate checks the last entry of that list and nothing else.

`musl` is a live third case, not a hypothetical: `isLinuxMusl()` selects `linuxmusl-x64.node`, a different filename on the same platform/arch.

Affected sites:
- `test/playwright/playwright.config.unit.mjs:111` — the hardcoded artifact path
- `test/playwright/playwright.config.unit.mjs:83` — `BRAIN_TIER_SETUP_GUIDANCE`, whose remediation is `npm rebuild better-sqlite3`; under v13 that command cannot fix this failure because it is not what is broken
- `test/playwright/unit/playwrightConfigUnit.spec.mjs:18` — asserts the guidance string contains that command, so the contract is already pinned and moves with the fix
- `.github/workflows/brain-unit.yml` step `Build the native SQLite binding` — still correct (it is v12's path and a source-build fallback), but its name now overstates what it does on a prebuild host

## The Fix

Make the `better-sqlite3` row prove *the binary the loader will actually load*, not one of the three places it may live:

- Keep `lib/index.js` as a required entrypoint.
- Replace the single `build/Release/better_sqlite3.node` requirement with an **alternatives** set satisfied by any one of `prebuilds/<target>.node` (target computed the way `getPrebuildPath` computes it, `linuxmusl` included), `build/Release/better_sqlite3.node`, `build/Debug/better_sqlite3.node`. This is version-agnostic: v12 satisfies it via `build/Release`, v13 via `prebuilds`.
- ~~Prefer delegating to the package's own `getPrebuildPath()` when the installed version exports it.~~ **Rejected during implementation, deliberately.** Delegation is circular: the gate exists to answer *"is this package installed?"*, and importing from that package to find out fails exactly when the answer is no. It must also stay sync and non-throwing before collection. And it removes nothing — `getPrebuildPath()` returns `null` when no prebuild matches, so the `build/` fallbacks are needed either way. The target string is mirrored instead (`nativeSqliteArtifacts`), with upstream named in the JSDoc as the authority and the arms pinning the target strings against literals so a convention change reds here rather than drifting.
- Update `BRAIN_TIER_SETUP_GUIDANCE` so its remediation leads with `npm ci` — the fix in both layouts — instead of implying the rebuild is what repairs a prebuild-shipping install.

`hasBrainTier` takes `rootDir`, so the shape stays one row in one table.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `hasBrainTier(rootDir)` `better-sqlite3` row | `better-sqlite3/lib/binding.js#getBinding` search order | admit when `lib/index.js` **and** any loader-reachable binary exist | path set when `getPrebuildPath` is unexported (v12) | gate `@summary` | `playwright.config.unit.mjs:109-116` |
| `BRAIN_TIER_SETUP_GUIDANCE` | this gate | remediation true for prebuild **and** source-built installs | — | inline | `playwright.config.unit.mjs:83`; spec `:17-20` |
| `npm rebuild better-sqlite3` as remediation | v13 `binding.gyp` `prebuild_exists` | no longer the fix for this failure; keep for the source-build case | — | guidance string | upstream comment, quoted above |

## Decision Record impact

`none`. A staleness fix inside one existing helper.

## Acceptance Criteria

- [ ] Red-first: an arm reproduces the false negative — a `better-sqlite3` package directory carrying `lib/index.js` + `prebuilds/<host target>.node` and **no** `build/` is admitted by `hasBrainTier`, and fails against today's implementation.
- [ ] A genuinely partial tier still fails: `lib/index.js` present with **no** binary in any of the three locations is rejected. This is the arm that proves the loosening did not delete the gate.
- [ ] `build/Release`-only (v12's layout) is still admitted — no regression for the version on `dev`.
- [ ] `linuxmusl` target selection is covered, not just `<platform>-<arch>`.
- [ ] `BRAIN_TIER_SETUP_GUIDANCE` names a remediation that is true under both layouts. **Amended:** the original wording also asked the spec to stop asserting `` `npm rebuild better-sqlite3` `` verbatim. That was wrong — the command is still the remediation for versions that build locally, so the existing substring assertion stays a real guard and replacing it would be churn that pins nothing new. The arm is unchanged and still passes.
- [ ] `chromadb` and `@chroma-core/default-embed` rows are untouched — measured identical across both lockfiles, so nothing licenses widening them.
- [ ] Post-merge: `#357` (or its successor after `#358` splits the group) reaches step 8 instead of failing step 7.

## Out of Scope

- **`#358`'s grouping fix.** Independent: `#358` makes a tier move arrive readable, this makes the gate read it correctly. Both are needed and neither substitutes.
- **Whether to adopt `better-sqlite3@13`.** This ticket makes the decision *reviewable* by removing a false red from it. `#358` explicitly declines to judge v13 and so does this.
- **Renaming the `Build the native SQLite binding` workflow step.** Its name overstates what it does on a prebuild host — cosmetic, and not worth a workflow edit.

**Scope ADDED during implementation** (the original Out-of-Scope clause above said touching `brain-unit.yml` "would mix a workflow edit into a guard fix" — that reasoning was written about the cosmetic rename and does not survive contact with this):

- [ ] `test/playwright/unit/playwrightConfigUnit.spec.mjs` joins the smoke list in `.github/workflows/brain-unit.yml`. **Measured:** CI runs `npm run test-unit -- --list` (collection only) plus **seven named specs**, and this file is not one of them — so every arm in this PR would be a guard that never executes where the false negative actually happened. The workflow states the criterion itself, one line above the list: *"A cross-PR drift detector is worthless outside this list, so it belongs in it."* These arms are exactly that detector, for exactly the drift that produced `#357`'s red. Shipping the fix without this would leave the next layout change as undetectable as this one was.
- **The `allowScripts` warning.** Present in green and red runs alike; nothing to fix.

## Avoided Traps

⛔ **Do not "fix" this by pinning back to `12.11.1`.** v13's model is strictly better for this repository — the binary arrives in the tarball, so CI neither downloads nor compiles it, which is what the exact pin was protecting. Pinning back would preserve a gate bug as a dependency policy.

⛔ **Do not replace the husk check with `require.resolve`.** The gate's `@summary` already rejects that and is right: resolution would admit a genuinely broken install. The defect is the hardcoded *location*, not the decision to prove an artifact.

⛔ **Do not chase the `allowScripts` warning.** It is identical in the green run. I pursued it first and it cost two calls to eliminate; it is recorded here so the next reader skips it.

⛔ **Do not conclude from "the module imports fine" that the tier is complete.** That is the exact instrument error `#358` recorded as a falsification. The gate deliberately asks a stronger question than the loader does.

## Related

- `#358` — the grouping half; its "Avoided Traps" note about `better-sqlite3@13.0.3` being falsified is superseded by this ticket and will be corrected in its body.
- `#359` — `#358`'s PR, which is what made this attributable.
- `#352` / `#355` — the dependabot rollout that surfaced it.
- `#357` — the red PR.
- `#201` — established the Brain unit suite in CI, which this gate protects.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `query_raw_memories("better-sqlite3 prebuilds hasBrainTier build/Release artifact path")` · commit-range anchor `origin/dev` vs `dependabot/npm_and_yarn/all-deps-4f991cb6ea` lockfile `prebuild-install` delta

Live latest-open sweep: checked latest 20 open issues at 2026-09-15T18:40:04Z; A2A claim sweep over the last 12 messages (all read-states); own-assignment sweep read `#358`'s body in full. No equivalent found — `#358` is the nearest and defers this explicitly.



## Timeline

- 2026-09-15T18:42:21Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T18:42:22Z @neo-opus-vega added the `bug` label
- 2026-09-15T18:42:23Z @neo-opus-vega added the `ai` label
- 2026-09-15T18:42:23Z @neo-opus-vega added the `testing` label
- 2026-09-15T18:42:23Z @neo-opus-vega added the `build` label
- 2026-09-15T19:00:19Z @neo-opus-grace cross-referenced by PR #357
- 2026-09-15T19:15:06Z @neo-opus-vega referenced in commit `e8c44de` - "fix(test): the tier gate admits the binary better-sqlite3 ships, not the one it used to build (#360)

`hasBrainTier` required `build/Release/better_sqlite3.node`. That file is produced by
`better-sqlite3@12`'s `install` script (`prebuild-install || node-gyp rebuild`), which
`13.0.3` deleted along with the `prebuild-install` dependency: v13 ships eight prebuilt
binaries in its tarball under `prebuilds/<target>.node` and never creates `build/`.
Upstream's `binding.gyp` states the intent — "npm's implicit node-gyp rebuild should do
nothing when the package contains a prebuild for the host" — so `npm rebuild
better-sqlite3` correctly built nothing in 4.2s and the gate read a healthy install as a
partial tier, failing brain#357 before collection.

`nativeSqliteArtifacts` now names every location the package's own loader tries, and a
requirement may be one path or an array of interchangeable ones. `chromadb`'s two
entrypoints stay SEPARATE requirements because a dependent runs each; the sqlite binary is
one requirement with several spellings because only one is ever loaded. `linuxmusl` is a
real third target, not a hypothesis.

The spec had no coverage of `hasBrainTier` at all. Six arms added, each mutation-verified
one row at a time — an `every` over three packages masks the second arm behind the first,
which made the chromadb control look dead when it is not.

Added beyond the ticket: this spec joins the CI smoke list. `brain-unit.yml` runs
collection plus seven named specs, so without that line every arm here would have guarded
nothing where the false negative actually happened — and the workflow says so itself one
line above the list.

Corroborated independently while this was in review: brain#361 carries the same twelve
updates with `better-sqlite3` untouched and its `unit` job passes, so the entire red/green
delta was this one dependency."
- 2026-09-15T19:16:06Z @neo-opus-vega cross-referenced by PR #363
- 2026-09-15T20:11:52Z @tobiu referenced in commit `1bff057` - "Merge pull request #363 from neomjs/agent/360-tier-gate-prebuilds

fix(test): the tier gate admits the binary better-sqlite3 ships, not the one it used to build (#360)"
- 2026-09-15T20:11:52Z @tobiu closed this issue

