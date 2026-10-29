---
id: 1342
title: generateRosterOnboarding still targets neo's README roster table
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-28T23:26:49Z'
updatedAt: '2026-10-29T00:14:39Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1342'
author: neo-opus-ada
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
closedAt: '2026-10-29T00:14:39Z'
---
# generateRosterOnboarding still targets neo's README roster table

## Context

Slice of #201. On Brain dev `8c718b21`, `generateRosterOnboarding.spec.mjs` › "a fresh resident plans MISSING on all four REAL onboarding surfaces" fails at `expect(planned.valid).toBe(true)`. The README surface is DIVERGENT and not repairable:

```
insertion anchor not found: the '| Name | Maintainer | Role | Identity |' table header is missing from README.md
```

It is one of dev's retained-suite reds (#201, comment 6482850355).

## The Problem

The generator came into the Brain with `11552b08` (2026-08-26) and kept neo's README shape. That shape is a four-cell row, `| - | [@login](…) | AI maintainer (<vendor> <family> family — engine designation pending first boot) | Machine Account |`. Its rotation half rewrites the model designation inside that row, because `neomjs/neo#14901` treated the README roster as a coherence mirror of ModelStats.

One day later, #186 (`44ccad15`) gave the Brain README its own roster, `| Name | Maintainer | Identity |`, with no model facts. The README states the rule under the table:

> Live participation and model embodiments change more often: `ai/graph/identityRoots.mjs` is the canonical identity/status registry, and `ModelStats.md` owns model facts. The README does not freeze either into a second authority.

A read-only probe of the real files at `8c718b21` shows both halves fail closed:

- **Onboarding:** the README surface is DIVERGENT (`repairable: false`), so `planOnboardingSurfaces` invalidates the whole payload for every new resident. `ai/scripts/fleet/onboardPeer.mjs:461` still prints the generator as an onboarding step.
- **Rotation:** a rotation of `@neo-gpt-emmy` to a new designation plans ModelStats as repairable, but the README half refuses: `README row does not contain the prior ModelStats designation 'GPT-5.6 Sol' exactly once`. So the rotation plan is invalid as well.

The spec's fixtures (`README_FIXTURE`, `ROTATION_FILES.readme`) still use neo's four-column table. Every fixture test passes, and only the live-file test sees the drift.

## The Architectural Reality

- `ai/scripts/setup/generateRosterOnboarding.mjs` is the only code that knows the README shape:
  - `renderReadmeRow` renders the row;
  - `planReadmeSurface` holds the header anchor and the generated-pending-row predicate (`includes('engine designation pending first boot') && endsWith('| Machine Account |')`);
  - `planRotationReadmeSurface` rewrites the designation;
  - the module docblock describes a `Name / Maintainer / Role / Identity` table, and `planRotationSurfaces` says "Only README + ModelStats can be writable".
- No other code parses the table: `git grep "| Maintainer |"` matches only this file and its spec. `onboardPeer.mjs` prints the CLI command, and `onboardPeer.spec.mjs` imports only `parseGenerateArgs`.
- Design authority: "The README does not freeze either into a second authority." (`README.md`, from #186, whose AC was that "no stale model/handle table is copied blindly").
- Decision Record impact: aligned-with `neomjs/neo#adr:0018-neo-identity-source-of-truth-model` §2.4, which prefers a reference to the single source over a hand-kept duplicate.
- Placement: `npm run ai:structure-map -- --files --loc` lists the file under `ai/scripts/setup`. No file is created or moved.

## The Fix

1. `renderReadmeRow` emits the Brain table's three cells: `| - | [@<login>](https://github.com/<login>) | AI maintainer |`. The Name cell stays `-` until a Social Name exists. Family and designation stay in identityRoots and ModelStats.
2. `planReadmeSurface` anchors on `| Name | Maintainer | Identity |` and recognizes its own pending row by shape: Name `-`, Identity `AI maintainer`. A login correction still repairs that row, and a named or hand-edited row still fails closed.
3. Rotation stops writing the README. `planRotationReadmeSurface` becomes a byte-stable boundary, like the roots, spec and migration halves:
   - MATCH when the resident has exactly one row and that row carries no designation;
   - DIVERGENT and not repairable when the row is missing, ambiguous, or still mirrors the prior designation. Under #186 such a row should not exist, so it is reported, not rewritten.

   `ROTATION_SURFACE_KEYS` keeps its five surfaces; only ModelStats is writable.
4. The docblocks follow: the module's surface list, `renderReadmeRow`, `planRotationReadmeSurface` and `planRotationSurfaces`.
5. Spec changes:
   - both README fixtures take the Brain shape;
   - the row, duplicate-row, rotation and CLI-rotation tests follow;
   - one test pins the refusal of a designation-mirroring row;
   - one live-file test pins rotation against the REAL README.

## Acceptance Criteria

- [ ] AC-1: "a fresh resident plans MISSING on all four REAL onboarding surfaces" passes under the unit config with `--retries=0`, and fails on dev `8c718b21`.
- [ ] AC-2: The generated row is the three-cell Brain shape with no family or designation text. Applied to the real README, it lands inside the roster table, and a rerun reports MATCH.
- [ ] AC-3: Rotating a real resident against the real README plans the README surface as MATCH with `updated: null`. In the fixture CLI, `--mode rotation --write` writes ModelStats only (one `[WROTE]`), and README.md stays byte-equal.
- [ ] AC-4: A README row that still carries the prior designation is DIVERGENT and not repairable, and the reason names the README's no-model-facts rule.
- [ ] AC-5: Brain Unit CI lists this spec among `fixed`, with `0 introduced`.

## Out of Scope

- The other #201 reds: `graphBackup`, `GuideToolParity` and `lintGuides`.
- Rotation for renamed seats: ModelStats keeps pre-rename anchors (`§neo_opus`, `§neo_gpt`, `§neo_gemini_pro`) while the generator derives the anchor from the current handle. That's a separate drift, captured as a defect-note.
- neo's README, which keeps its four-column table; this generator does not write it.
- What the README lists, and the `onboardPeer.mjs` flow.

## Avoided Traps

- **Anchor-only fix:** it would find the header but append a four-cell row to a three-column table, and keep writing model facts that #186 removed.
- **Accepting both table shapes:** this would keep a neo branch alive that no Brain caller reaches.
- **Dropping README from `ROTATION_SURFACE_KEYS`:** this would change the five-surface CLI and report contract for no gain. A boundary keeps the contract and checks the invariant.
- Kept from `neomjs/neo#14954`: "Do not treat substring presence as ownership." The pending row is recognized by its exact shape.

## Related

Slice of #201. The drift came in with `11552b08` (generator) and `44ccad15` (#186, README). Prior rotation design: `neomjs/neo#14901` / PR `neomjs/neo#14902`; ceremony contract: `neomjs/neo#14954`.

Live latest-open + closed sweep: checked the latest 20 open Brain issues at 2026-10-28T23:25:53Z, plus closed searches for "roster README onboarding generator" and "generateRosterOnboarding" (Brain) and "generateRosterOnboarding README" (neo). Read the Out of Scope and Avoided Traps of the same-surface closed hit `neomjs/neo#14954`. No equivalent ticket or prior exclusion found.
A2A sweep: the 30 most recent messages (all read-states) hold no lane-claim on this surface.
MC sweep: "roster onboarding README table header missing insertion anchor not found new resident cannot be onboarded", 8 results, no prior decision found.
Own-assignment sweep: 17 open; only the parent #201 overlaps.

Ownership: self-assigned to @neo-opus-ada. This is the next slice of my in-flight #201 burn-down, so it isn't offered as a good first issue.

Origin Session ID: 6cfc6a71-3d75-4616-87fc-b8ba06833824
Retrieval Hint: "generateRosterOnboarding README roster table drift after Brain #186"

## Timeline

- 2026-10-28T23:26:51Z @neo-opus-ada added the `bug` label
- 2026-10-28T23:26:51Z @neo-opus-ada added the `ai` label
- 2026-10-28T23:26:51Z @neo-opus-ada added the `testing` label
- 2026-10-28T23:26:56Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-28T23:34:37Z @neo-opus-ada cross-referenced by PR #1343
- 2026-10-29T00:14:39Z @tobiu referenced in commit `b98d30e` - "fix(setup): the roster generator writes the Brain README's three-column row (#1342) (#1343)

The generator kept neo's four-cell README row and rotated a model designation inside it. The Brain README has its own three-column roster with no model facts, so onboarding could not find its anchor and every real rotation failed closed on the README half. Rotation now treats the README as a byte-stable boundary and refuses a row that still mirrors a designation."
- 2026-10-29T00:14:40Z @tobiu closed this issue
- 2026-10-29T00:15:59Z @neo-opus-ada cross-referenced by #201

