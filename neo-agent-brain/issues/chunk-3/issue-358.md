---
id: 358
title: 'The Brain tier rides the bulk bump, so a major native move arrives camouflaged'
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-09-15T18:15:09Z'
updatedAt: '2026-09-15T18:54:50Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/358'
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
closedAt: '2026-09-15T18:54:50Z'
---
# The Brain tier rides the bulk bump, so a major native move arrives camouflaged

## Context

`#352` / `#355` added this repository's first `.github/dependabot.yml`. Its **first** `all-deps` PR is `#357`, and it is red:

```
[playwright.config.unit] CI requires the complete Brain tier (better-sqlite3,
chromadb, @chroma-core/default-embed) but it is absent or partial — a skipped
brain matrix on a green CI run is silent coverage loss.
```

`#357` bumps **13 dependencies in one PR**, three of them effective majors — `better-sqlite3` 12.11.1 → **13.0.3**, `chalk` ^5 → ^6, `catharsis` 0.9.0 → 0.11.0 — and prunes the lockfile from **295 to 261** packages.

**This is my config's defect, not upstream's.** I wrote the exclusion rationale for `neo-agent-skills` in that same file:

> *"Excluded so its diff has to be read: a standalone PR is the only place that happens, and inside the bulk bump it gets skimmed."*

And in `#352`'s own body I observed that the tier packages are *"exact-pinned, which is correct for reproducibility"* — then left them inside `all-deps` anyway. **Same argument, applied to one dependency and not the others.**

## The Problem

A **major native-module bump wrapped in twelve cosmetic ones** is exactly the diff the exclusion rationale exists to prevent. The consequences are visible in `#357`:

1. **One verdict for thirteen decisions.** `#357` cannot be approved or rejected on its merits — `cors 2.8.5 → 2.8.6` and a major rebuild of the SQLite binding share a single review and a single red.
2. **The reproducibility pin is defeated in effect.** These three are exact-pinned because the Brain tier must be reproducible. A bulk bump moves them with the same ceremony as a patch-level dev dependency.
3. **The failure was unattributable.** I could not determine which of the thirteen made the tier partial, which is itself the cost of grouping. **Resolved since, in `#360`:** `better-sqlite3@13` ships its binary in the tarball (`prebuilds/<target>.node`) and stopped creating `build/Release/better_sqlite3.node`, which is the only path `hasBrainTier` probes. The tier is complete; the red is a false negative from a v12-shaped gate. My ruling-out below was the wrong instrument — see the correction in Avoided Traps.

## The Architectural Reality

`exclude-patterns` removes a package from the **group**, not from updates — it still gets a PR, its own one. **Verified empirically in this same rollout:** `neo-agent-institution#146` arrived as a standalone `monaco-editor` bump because that repository's config excludes it. The mechanism is proven in the org, not assumed from documentation.

The three tier packages are what `playwright.config.unit`'s gate names, so they are already recognised elsewhere as a coupled set that governs whether the Brain matrix can run at all. A dependency that can silently disable a test tier belongs in a reviewed PR by the same logic that put `neo-agent-skills` there — that one rewrites the workflows agents obey; these decide whether the Brain's tests execute.

## The Fix

```yaml
exclude-patterns: ["neo-agent-skills", "better-sqlite3", "chromadb", "@chroma-core/default-embed"]
```

Record the tier rationale beside the existing skills one: these are exact-pinned for reproducibility and gate whether the Brain matrix runs, so each moves in a PR whose diff gets read.

`#357` then closes and reopens split — a smaller `all-deps` group plus one PR per tier package, each judgeable on its own.

## Decision Record impact

`none`. Widening an existing exclusion list on the reasoning the file already records.

## Acceptance Criteria

- [ ] `.github/dependabot.yml` excludes all three tier packages from `all-deps`, with the rationale recorded in the file beside the `neo-agent-skills` note.
- [ ] The file states that `exclude-patterns` means **its own PR**, not "no updates" — the distinction the whole design rests on, and the one a future reader is most likely to invert.
- [ ] The config parses (`js-yaml`), and the exclusion list contains no package absent from `package.json` — a pattern naming an absent package reads as a governing rule while governing nothing.
- [ ] `neo.mjs`'s tarball-URL carve-out comment is unchanged and still accurate.
- [ ] Post-merge: the next `all-deps` PR carries none of the three, and any tier move arrives as its own PR.

## Out of Scope

- **`#357` itself.** It stays open or gets superseded by the split; this ticket does not judge `better-sqlite3@13`.
- **Why the tier went partial.** Was unresolved here; now owned by `#360` — the gate probes `build/Release/better_sqlite3.node`, which `better-sqlite3@13` never creates because it ships prebuilt binaries instead. Independent of this ticket: `#360` makes the gate read a tier move correctly, this makes the move arrive readable. Neither substitutes for the other.
- Other repositories' configs. `neomjs/neo` excludes `monaco-editor` and `neo-agent-skills` and has no Brain tier; `devindex` and `neo-agent-institution` likewise. This is Brain-specific.
- `#300` / sharp — a specific vulnerable dependency, owned there.

## Avoided Traps

⛔ **`exclude-patterns` does not mean "never update".** It removes the package from the *group*, so it still gets a PR — a solo one. Reading it as an opt-out is the inversion that would make this change look like pinning-by-neglect, and it is why the AC requires the file to say so.

**Do not add a `semver-major` ignore instead.** It would stop the noise and also stop the notification, which is the opposite of the goal: the point is that a major native bump gets *read*, not that it never arrives.

⛔ **CORRECTED — my falsification of `better-sqlite3@13.0.3` was the wrong instrument, and `#360` has the cause.** This ticket originally said the suspect "is falsified — it installs and runs standalone". That measurement was real and answered a different question: whether the **module loads**, never whether the **artifact `hasBrainTier` probes** exists. It could not have produced the failing result, so it never tested the gate's claim. `better-sqlite3@13` was in fact the right suspect — not because it is broken, but because it ships `prebuilds/<target>.node` and no longer creates `build/Release/better_sqlite3.node`. Leaving the original sentence standing would have sent the next reader away from the answer.

**Do not infer the tier failure's cause from this ticket.** `#360` owns it; read that.

## Related

- `#352` / `#355` — the config this corrects; the rationale it under-applied is quoted there.
- `#357` — the first `all-deps` PR and the live evidence.
- `neo-agent-institution#146` — the standalone-PR mechanism, observed working.
- `neo-agent-skills#14` — cross-repo governance, if exclusion policy is ever unified.

**Sweeps.** Live latest-open: newest 10 open issues here at 2026-09-15T18:14:27Z (#349 … #282) plus an exact `dependabot group exclude` search across all states — only `#352`, closed, which is the config this amends. A2A in-flight claim sweep over the latest 30 messages, all read-states: @neo-opus-grace is watching `#356`/`#357` CI and explicitly holds no claim on the config; no competing claim. Memory Core rationale sweep on the problem's nouns: no prior decision on group scope. Own-assignment sweep: `#342` is mine here and unrelated.

Origin Session ID: a2868f57-a008-4abf-b493-b87be1636964

Retrieval Hint: `brain dependabot all-deps group exclude better-sqlite3 chromadb tier reproducibility standalone PR`


## Timeline

- 2026-09-15T18:15:09Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-09-15T18:15:10Z @neo-opus-vega added the `bug` label
- 2026-09-15T18:15:11Z @neo-opus-vega added the `ai` label
- 2026-09-15T18:16:31Z @neo-opus-vega cross-referenced by PR #359
- 2026-09-15T18:54:50Z @tobiu referenced in commit `4208bca` - "Merge pull request #359 from neomjs/agent/358-tier-exclusions

The Brain tier leaves the bulk group, so its bumps arrive readable (#358)"
- 2026-09-15T18:54:50Z @tobiu closed this issue
- 2026-09-15T19:00:19Z @neo-opus-grace cross-referenced by PR #357
- 2026-09-15T19:16:06Z @neo-opus-vega cross-referenced by PR #363

