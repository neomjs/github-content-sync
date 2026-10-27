---
id: 1096
title: Shared-config writes in a test file may not exceed its ceiling
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-18T02:19:53Z'
updatedAt: '2026-10-18T03:14:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1096'
author: neo-opus-grace
commentsCount: 0
parentIssue: 1091
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-18T03:12:42Z'
---
# Shared-config writes in a test file may not exceed its ceiling

## Context

#1093 (head `fc078f0`) added two `aiConfig.issueSync.contentRootOverride` writes to `SyncService.Stage2.spec.mjs`, a spec with zero shared-config writes on `dev`, and CI stayed green. `check-aiconfig-test-mutation` counted 577 at the merge base `54cd8fc4` and 579 at the head, and printed both. It does not fail, because the full B4 scope is report-only (ADR-0019 §3 B4, #508). Review caught it ([5561011420](https://github.com/neomjs/neo-agent-brain/pull/1093#pullrequestreview-5561011420), RA-1); no gate did.

#541 and #1091 AC-3 defer global gating until the count reaches 0. Until then, nothing stops new debt. @neo-gemini-pro, who owns #1091, chose this leaf over folding it into #1091, and set its constraint: it must not depend on merge order.

## The Problem

The report-only rule makes the surface visible but leaves it unbounded. Any spec, clean or legacy, can add writes and stay green, so the conversions under #1091 race against new writes landing elsewhere. A reviewer who has not read ADR-0019 sees a green guard.

## The Architectural Reality

- `ai/scripts/lint/check-aiconfig-test-mutation.mjs`: `ADR_0019_RULES` holds two B4 rules. `db-path-subset` gates; `full` (`findSharedConfigMutations`) is report-only. `main()` makes one exit decision over the DB-path and restore-capture violations.
- At `origin/dev` `1b49965d`: 545 shared-config writes in 53 of 929 test files, by the guard's own count. `SyncService.Stage2.spec.mjs` has 0.
- Sibling lints keep their registries as JSON beside the script (`config-leaf-parity.json`, `retry-bound-registry.json`). The owning folder is `ai/scripts/lint` (structure map: 22 files).
- #508 and #541 rejected per-leaf escape markers because they silence the detector. A per-file count ceiling silences nothing: every write is still detected and reported, and only growth fails.

## The Fix

1. `ai/scripts/lint/shared-config-write-ceilings.json` records, per test file, its shared-config writes at the seed tree. A file absent from it has ceiling 0.
2. The guard fails when a file's count exceeds its ceiling, inside the same exit decision as the DB-path and restore-capture classes. It reports files below their ceiling without failing.
3. `--tighten-ceilings` rewrites the sidecar to min(ceiling, current count) and drops zeros. It never raises a ceiling and never adds a file. Raising one is a hand edit a reviewer sees.
4. ADR-0019 §3's B4 row and the guard's report-only rationale name the ratchet.

**Merge-order independence.** Conversions only lower counts, so #1091 and this leaf stay green in either order. Whoever lands second runs `--tighten-ceilings`.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `check-aiconfig-test-mutation` exit code | ADR-0019 §4 B4; #508 | Also nonzero when a test file's shared-config writes exceed its ceiling | At or below the ceiling: report-only, as today | the guard's JSDoc; ADR-0019's B4 row | unit; a CLI red on the #1093 shape |
| `shared-config-write-ceilings.json` | this ticket | Per-file ceilings at the seed tree; an absent file has ceiling 0 | — | the guard's JSDoc | the seed tree passes |
| `--tighten-ceilings` | this ticket | Lowers to the current count, drops zeros; never raises or adds | — | the guard's JSDoc | unit |

**Decision Record impact:** amends ADR-0019 §3, whose B4 row gains the per-file ceiling. Aligned with §4–§5: isolate by construction, no escape markers.

## Acceptance Criteria

Delivered by #1097, merged at `a9bd033822` (2026-10-18T03:12Z).

- [x] AC-1: On the seed tree, the guard exits 0 and still prints the full count. The full scan prints `545 shared-config write(s)` and exits 0, and the sidecar equals the live per-file counts: 53 entries, 0 mismatches.
- [x] AC-2: A file at 0 that gains a write fails the guard, and the message names the file, its count and its ceiling. This is the #1093 shape, covered by a unit arm and a CLI receipt. #1093's original spec, taken from `fc078f0`, exits 1 with `SyncService.Stage2.spec.mjs: 2 write(s), ceiling 0`. Unit arm: "a file the ceilings do not name holds none". CLI arm: "a clean spec that gains shared-config writes fails the build".
- [x] AC-3: A file at or below its ceiling passes. Files below their ceiling are reported. Unit arm: "a file at or under its ceiling passes; one above it breaches with both numbers". Files below their ceiling print a report line naming `--tighten-ceilings`.
- [x] AC-4: `--tighten-ceilings` lowers and drops entries; it never raises or adds (unit). Unit arm: "tightening lowers to the measured count and drops zeros, but never raises or adds". The flag also refuses a partial scan, which has its own CLI arm.
- [x] AC-5: ADR-0019's B4 row and the guard's rationale describe the ratchet. The B4 row says the clause "reports its count and gates its growth", the Status line carries the 2026-10-18 amendment, and `SHARED_CONFIG_MUTATION`'s rationale names the ceiling. `lint-config-template-ssot` reports 0 ownership mismatches.

The first tighten after a conversion is #1097's Post-Merge item, owned by #1091. #1099 is the first conversion to land after the ratchet.

## Out of Scope

- Converting any spec: that is #1091.
- Global gating at 0 (#1091 AC-3). When every ceiling reaches 0 the sidecar is empty, and the ratchet is that gate.
- Writes under `ai/**`. The guard scans `test/` only (ADR-0019 §4).

## Avoided Traps

- **A per-file allowlist of exempt files.** It would exempt any number of new writes in a listed file. A ceiling exempts none beyond today's count.
- **Failing when a file drops below its ceiling.** That forces #1091 to edit the sidecar in lockstep, which is exactly the merge-order coupling its owner ruled out.

## Related

#1091 (parent) · #1093 · #541 · #508 · #1092 (the `configSnapshot` seam)

Live latest-open sweep: checked the latest 20 open issues at 2026-10-18T02:19:00Z; none is equivalent (#1091 is the parent).
A2A claim sweep: the last 30 `lane-claim` messages, all read states; no claim on the guard.
Memory Core sweep: "a test spec gained shared-config writes and CI stayed green because the B4 full-scope rule is report-only". It returned the #1091 owner's endorsement of this leaf and #508's history; no prior decision against a ceiling.
Own-assignment sweep: #684, #18, #54; none overlaps.

Origin Session ID: bab4a460-7586-4505-b324-56756d2694e5
Retrieval Hint: "B4 shared-config writes per-file ceiling ratchet check-aiconfig-test-mutation"


## Timeline

- 2026-10-18T02:19:54Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-18T02:19:54Z @neo-opus-grace added the `enhancement` label
- 2026-10-18T02:19:54Z @neo-opus-grace added the `ai` label
- 2026-10-18T02:19:55Z @neo-opus-grace added the `testing` label
- 2026-10-18T02:20:04Z @neo-opus-grace added parent issue #1091
- 2026-10-18T02:32:52Z @neo-opus-grace cross-referenced by PR #1097
- 2026-10-18T02:34:47Z @neo-gemini-julia cross-referenced by #1098
- 2026-10-18T02:38:12Z @neo-gemini-julia cross-referenced by PR #1099
- 2026-10-18T03:12:43Z @tobiu closed this issue
- 2026-10-18T03:12:43Z @tobiu referenced in commit `a9bd033` - "feat(lint): a test file may not exceed its shared-config write ceiling (#1096) (#1097)"
- 2026-10-18T06:05:13Z @neo-gemini-julia cross-referenced by PR #1116

