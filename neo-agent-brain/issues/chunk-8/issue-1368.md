---
id: 1368
title: A Brain script run through a symlinked path exits 0 without running
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T00:07:33Z'
updatedAt: '2026-10-30T00:47:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1368'
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
closedAt: '2026-10-30T00:47:40Z'
---
# A Brain script run through a symlinked path exits 0 without running

## Context

Measured 2026-10-29 on `ai/scripts/lint/lint-guides.mjs`:
- Through the real path it prints its report (10 full guides and 63 guide-series files, 0 hard) and exits 0.
- Through a symlink to the same checkout it writes 0 bytes and exits 0. The defect-note is `a3a6ffba`.

The engine fixed this class in neomjs/neo#18931. The Brain never received that helper.

## The Problem

Node resolves symlinks in the entry module's `import.meta.url` and never in `process.argv[1]`. An entry check that leaves either side unresolved is false for a script run through a symlinked path. The script then skips its CLI block and exits 0:
- A lint or guard reports green having checked nothing.
- A tool or daemon does nothing, without saying so.

Where it hits: a checkout under macOS `/tmp` or `/var` (both are links into `/private`), and any absolute path through a link. The case that matters most is evidence. A base run in a `/tmp` scratch clone is exactly the "same instrument on base" that a PR cites. CI and `npm run` invoke by relative path through real directories, so they are not affected.

The Brain has already paid for this locally, three times:
- Symlink-projected hooks ran silently dead, so `projectSeatHooks.mjs` realpaths its check and copies hooks instead of linking them.
- `prepare.mjs` carries its own realpath check, with its own comment.
- The neural-link `standalone.mjs` launcher does the same, because `npx` runs it through a `bin` link.

## The Architectural Reality

At `dev@b6f24069fe`, 142 files under `ai/` read `process.argv[1]` (147 lines):
- **140 gate a CLI block on it.** 135 of those compare it with the module unresolved, in about a dozen spellings. The commonest are `import.meta.url === pathToFileURL(process.argv[1]).href` and `path.resolve(process.argv[1]) === __filename`. A few hoist the path into a local first (`cliEntryPath`, `invokedPath`, `currentFile`). `probePortClaims.mjs` matches `endsWith('probePortClaims.mjs')`, which survives a link but matches any file of that name.
- **The other 5 realpath it.** `standalone.mjs` and `packNeuralLink.mjs` resolve both sides. `projectSeatHooks.mjs`, `seatProjectionCheck.mjs` and `prepare.mjs` resolve `process.argv[1]` only, and compare it against `import.meta.url` as is, which `--preserve-symlinks-main` leaves unresolved.
- **2 files are not entry checks.** `ai/examples/cloud-deployment/pre-push-hook.sh` runs a `node -e` one-liner that reads its own argument, and `scriptPlaneClosure.mjs` names the old spelling in a docblock.

Outside `ai/`, `test/playwright/compareUnitFailures.mjs`, the CLI behind Brain Unit's head-vs-base comparison, carries the same check.

Two readers parse these checks:
- **`scriptPlaneClosure.mjs`.** Its `referencesEntryPath` tells a CLI block from code that runs on import. It accepts any `if` test that references `process.argv`, `import.meta` or a bound entry-path local, so `isEntryModule(import.meta.url)` still reads as guarded. Its docblock names the old spelling as "the house pattern".
- **`lint-config-template-ssot.mjs`.** Its `isThreadEntrypoint` decides which modules ADR 0019's C1 rule exempts as entrypoints, and it requires `process.argv[1]` and `import.meta.url` in one `if` test. The bare helper call would therefore demote every migrated script, including `runAgent.mjs` and every daemon, which its spec pins as entrypoints.

## The Fix

- **`ai/services/shared/isEntryModule.mjs`:** `isEntryModule(importMetaUrl)` realpaths both sides. It returns false when `process.argv[1]` is missing or names nothing on disk. This is the engine's helper, ported. `ai/services/shared/` already holds the Brain's other cross-cutting primitives (`atomicFileWrite.mjs`, `boundedRetryGate.mjs`).
- **Every entry check under `ai/`, and `compareUnitFailures.mjs`, calls it.** Imports and locals that only the old spelling used are removed, and comments that described the old comparison follow.
- **`isThreadEntrypoint` also accepts `isEntryModule(import.meta.url)`.** A hand-rolled comparison still counts.
- **`scriptPlaneClosure.mjs`:** its docblock names the new spelling.

## Acceptance Criteria

- [ ] AC-1: a unit spec for the helper. Through the real path, a symlinked path and `--preserve-symlinks-main`, a probe's block runs. Imported by another module, and with no `process.argv[1]`, it does not.
- [ ] AC-2: run through a symlinked checkout path, `lint-guides.mjs` reports again. It is red on `dev` (0 bytes) and green at head.
- [ ] AC-3: `git grep -n 'process.argv\[1\]' -- ai` finds no entry check outside the helper. What remains is the `.sh` one-liner and comments that describe the spelling the readers still accept.
- [ ] AC-4: `isThreadEntrypoint` classifies `if (isEntryModule(import.meta.url))` as an entrypoint, and `runAgent.mjs` and every daemon still classify as one.
- [ ] AC-5: Brain Unit's head-vs-base comparison introduces no failure.

## Out of Scope

- Spec files that build synthetic sources containing `process.argv[1]`.
- One helper shared across repositories. The engine's copy stays where it is.
- The hook projection's transport. `projectSeatHooks.mjs` copies hooks because the old check died through a link. With `isEntryModule` that reason is gone, but whether links would now serve better is a separate decision.

## Related

neomjs/neo#18931 (the engine's fix), #250 and #317 (the projected-hooks history). Decision Record impact: aligned-with ADR 0019. C1's entrypoint predicate learns the helper's spelling, and what it classifies is unchanged. Structure map: `ai/services/shared/`, the cross-cutting primitives.

Ownership: self-assigned. I found it while restoring neo's guide lint and am doing it in this session.

Live latest-open + closed sweep: latest 20 open issues, plus a closed search for `symlink` and an all-state search for `argv realpath`, at 2026-10-30T00:07Z, with no equivalent. #250, #317 and #687 concern symlinks but not entry checks. A2A (last 30): no claim. Memory Core: Grace's 2026-08-31 note on symlink-projected hooks dying silently, and Emmy's 2026-08-26 note on the `import.meta.url` / `argv[1]` mismatch. Neither produced a Brain-wide ticket. Own open assignments: none on this surface.

Origin Session ID: be08763b-ad29-4a7b-8f90-a8e06b01b8de

Retrieval Hint: "Brain entry check symlinked path exits 0 isEntryModule realpath process.argv"


## Timeline

- 2026-10-30T00:07:33Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T00:07:35Z @neo-opus-ada added the `bug` label
- 2026-10-30T00:07:35Z @neo-opus-ada added the `ai` label
- 2026-10-30T00:07:35Z @neo-opus-ada added the `refactoring` label
- 2026-10-30T00:25:15Z @neo-opus-ada cross-referenced by PR #1369
- 2026-10-30T00:28:24Z @neo-opus-ada referenced in commit `0ed2838` - "test(fleet): the Antigravity launcher fixture carries isEntryModule (#1368)

The spec builds a runtime generation from the launcher's import closure, listed by hand. fleetMcpLauncher.mjs now imports ai/services/shared/isEntryModule.mjs, so the list names it too. A published runtime tree carries the whole repository, so only this fixture was short a file."
- 2026-10-30T00:28:25Z @neo-opus-ada referenced in commit `d906777` - "test(maintenance): offHostSync finds backup.mjs's CLI footer by its isEntryModule guard (#1368)

The arm located the footer by the old guard spelling. Once that string was gone, indexOf returned -1 on both sides, and the overlay-load-first assertion failed. It now anchors on the new guard and fails by name if the anchor goes missing."
- 2026-10-30T00:47:40Z @tobiu referenced in commit `98603a4` - "fix(scripts): a Brain script run through a symlinked path runs its CLI block again (#1368) (#1369)

* fix(scripts): a Brain script run through a symlinked path runs its CLI block again (#1368)

ai/services/shared/isEntryModule.mjs realpaths both process.argv[1] and
the module before comparing them, so a script run through a symlinked path
(a macOS /tmp checkout, a package bin) no longer skips its CLI block and
exits 0 having checked nothing. Every entry check under ai/, and
compareUnitFailures.mjs, calls it.

isThreadEntrypoint, the entrypoint predicate behind ADR 0019's C1 rule,
accepts the helper call. Without it the migration would have demoted
runAgent.mjs and every daemon. scriptPlaneClosure already reads the call
as guarded, and its docblock now names the new spelling.

In touched files, comments that described the old comparison follow it.
Pre-existing comment archaeology is reworded or carries the typed marker.
Block alignment is repaired where the gate requires it.

* test(fleet): the Antigravity launcher fixture carries isEntryModule (#1368)

The spec builds a runtime generation from the launcher's import closure, listed by hand. fleetMcpLauncher.mjs now imports ai/services/shared/isEntryModule.mjs, so the list names it too. A published runtime tree carries the whole repository, so only this fixture was short a file.

* test(maintenance): offHostSync finds backup.mjs's CLI footer by its isEntryModule guard (#1368)

The arm located the footer by the old guard spelling. Once that string was gone, indexOf returned -1 on both sides, and the overlay-load-first assertion failed. It now anchors on the new guard and fails by name if the anchor goes missing."
- 2026-10-30T00:47:40Z @tobiu closed this issue

