---
id: 300
title: The substrate-size guard's Claude row reports its measured bound
state: CLOSED
labels:
  - enhancement
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-11-06T18:01:50Z'
updatedAt: '2026-11-06T18:29:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/300'
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
closedAt: '2026-11-06T18:29:42Z'
---
# The substrate-size guard's Claude row reports its measured bound

## Context

neomjs/neo#17175 measured Claude Code's `@`-import loading on 2026-11-06 ([receipt](https://github.com/neomjs/neo/issues/17175#issuecomment-6691663838)). CLI 2.1.212 loaded every marker of a `CLAUDE.md` that `@`-imports files of 20,000, 30,000 and 64,000 B. So it loads at least 64,000 B per import and at least 114,112 B per file; both figures are lower bounds. The Desktop host path, which most Claude seats run, was not measured. On 2026-11-06, @neo-opus-grace decided the row's shape: the check and the flag stay, and only the label changes.

## The Problem

The guard's `.claude/CLAUDE.md` row prints `limit inherited from another harness, semantics unconfirmed`, and the JSDoc on `TARGET_FILES` says the experiment "has not run". Both are now stale. The CLI half has been measured, and the unknown that remains is the Desktop `@`-import path. Someone reading the baseline output learns neither fact from the generic label.

## The Architectural Reality

- `scripts/check-substrate-size.mjs:36-51` holds `TARGET_FILES`, the row `{path: '.claude/CLAUDE.md', harness: 'Claude Code', limitConfirmed: false}`, and the JSDoc sentence.
- `scripts/check-substrate-size.mjs:270` appends the generic label to every unconfirmed row.
- In `neo` and `neo-stage`, `.claude/CLAUDE.md` is a symlink to `AGENTS.md`, so the Antigravity budget binds the same bytes. `create-app` writes `CLAUDE.md` as `@AGENTS.md` into outside workspaces, which makes Desktop plus an `@`-import a live path.

Design authority: "The guard's Claude row states the tested bound and the remaining gap instead of an assumed cap" and "The Claude row stays checked at the shared Antigravity budget. … `limitConfirmed` stays false until the Desktop host is measured, and the row's note prints the CLI bound and the gap" (neomjs/neo#17175, AC-2 and the open-question summary of its body as @neo-opus-grace narrowed it on 2026-11-06).

## The Fix

- `TARGET_FILES` rows take an optional `note`. The report prints a row's note in place of the generic label.
- The Claude row's note reads: `CLI 2.1.212 loads ≥114,112 B per file, ≥64,000 B per @-import (neomjs/neo#17175); Desktop @-imports unmeasured; checked at the shared Antigravity budget`.
- `limitConfirmed` stays `false`, and the row stays checked at `PER_FILE_LIMIT_BYTES`.
- The JSDoc names the CLI measurement and the Desktop gap that remains, replacing "has not run".
- The package version is bumped.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| The `check-substrate-size` report row for `.claude/CLAUDE.md` | neomjs/neo#17175's receipt; @neo-opus-grace's decision of 2026-11-06 | Prints the measured bound, the Desktop gap and the budget that binds | An unconfirmed row without a `note` keeps the generic label | JSDoc on `TARGET_FILES` | A `test-substrate-size.mjs` case |

## Acceptance Criteria

- [ ] The `.claude/CLAUDE.md` row prints the note above. It is still checked at 23,800 B, and `limitConfirmed` is still `false`.
- [ ] An unconfirmed row without a note still prints the generic label.
- [ ] The `TARGET_FILES` JSDoc no longer says the experiment has not run; it names the CLI bound and the Desktop gap.
- [ ] The package version is bumped.

## Out of Scope

- The Desktop `@`-import measurement, which neomjs/neo#17175 keeps.
- Any change to the numeric limit, or a separate Claude limit. #242's Out of Scope stands.
- The turn-memory pre-flight payload. It does not mention the budget, and the guard's row is the one place this fact lives.

## Related

neomjs/neo#17175 · #242

Decision Record impact: none.

Live latest-open and closed sweep at 2026-11-06T18:01Z: I checked the latest 20 open Skills issues and ran a closed search for "substrate size Claude limitConfirmed". It found #242, and I read its Out of Scope. Nothing equivalent exists, and no prior exclusion applies.
A2A claim sweep (30 messages, all read states): no claim on this scope. Grace handed it to me at 17:59Z.
MC sweep ("substrate-size guard Claude row … @-import budget"): only #17175's own trail.
Own-assignment sweep: #5 and #6, neither overlapping.
Ownership: self-assigned for immediate delivery, so it is not a `good first issue` reservation.

Origin Session ID: 928dfeec-33e1-40d5-967c-1081353a09af

Retrieval Hint: "check-substrate-size Claude row note measured @-import bound Desktop gap"

Authored by Vega (Opus 5.5, Claude Code) 🌿


## Timeline

- 2026-11-06T18:01:50Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-06T18:01:52Z @neo-opus-vega added the `enhancement` label
- 2026-11-06T18:01:52Z @neo-opus-vega added the `ai` label
- 2026-11-06T18:04:15Z @github-actions added the `needs-design-authority` label
- 2026-11-06T18:05:31Z @neo-opus-vega cross-referenced by PR #301
- 2026-11-06T18:22:32Z @github-actions removed the `needs-design-authority` label
- 2026-11-06T18:29:42Z @tobiu referenced in commit `439b842` - "feat(guard): the Claude row of check-substrate-size reports its measured bound (#300) (#301)

TARGET_FILES rows take an optional `note`, printed in place of the generic
"limit inherited from another harness, semantics unconfirmed" label. The
.claude/CLAUDE.md row's note carries neomjs/neo#17175's CLI measurement
(at least 114,112 B per file and 64,000 B per @-import) and the unmeasured
Desktop @-import path. The row stays checked at the shared Antigravity
budget, and limitConfirmed stays false. run() gains a `targets` test seam,
mirroring collectReport's. Version 0.1.95."
- 2026-11-06T18:29:43Z @tobiu closed this issue

