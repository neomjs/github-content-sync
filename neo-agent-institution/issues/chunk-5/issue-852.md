---
id: 852
title: The visual stamp records the running engine even when no golden changed
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - testing
assignees:
  - neo-gemini-maryam
createdAt: '2026-10-31T18:41:05Z'
updatedAt: '2026-10-31T19:06:55Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/852'
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
closedAt: '2026-10-31T19:06:55Z'
---
# The visual stamp records the running engine even when no golden changed

## Context

`npm run stamp-visual-baselines` rewrote the `engine` line of `test/playwright/visual/__screenshots__/baseline-inputs.txt` in two pull requests that captured no golden:

- PR #850 changed banner code. Its author (Ada) put the line back to `95e63df6…` by hand before merge.
- PR #851 changes one cadence constant and two doc comments. At head `f35989d` the line moved from `95e63df6…` to `990669e5…`, which is the author's locally installed engine (review RA-3).

These are two independent occurrences of the same trap, and the trap fires every time a PR touches a style-owning input without re-capturing.

## The Problem

Design authority: #646's visual-stamp ruling (operator, 2026-10-09: "track dev, the stamp notes drift"). The stamp records the engine revision its goldens were **captured** against. The freshness check prints `goldens captured at engine X, running Y` without failing, so a red local capture run can be traced to the engine.

`--stamp` does not keep that meaning. It always writes the engine installed on the stamping machine. So an input-only stamp moves the line to an engine nobody captured at. After that, `engineDrift()` compares against the wrong revision: a capture failure caused by the engine's own change since the real capture no longer shows the notice, and reads as the PR author's fault.

## The Architectural Reality

- `buildScripts/checkVisualBaselines.mjs`: `current = {engine: engineRevision(), entries}`, and `--stamp` writes `serializeStamp(current)` unconditionally. `engineRevision()` reads the installed `neo.mjs` from the local lock.
- The input scopes include the golden sets themselves (the file header: "the two capture specs and their golden sets"). Whether a golden changed in this stamp is therefore visible by comparing `current.entries` with the parsed previous stamp.
- `parseStamp` / `serializeStamp` / `diffStamp` / `engineDrift` are the pure, import-safe halves; the git-touching main flow runs only as the entry script.

## The Fix

`--stamp` keeps the previous stamp's `engine` value unless a golden file's blob changed in this stamp. When a golden changed, a capture happened, so it records `engineRevision()`. A first stamp, with no stamp file yet, records `engineRevision()`. Put the decision in a pure helper beside `engineDrift` (for example `nextStampEngine(previous, current)`) so it is unit-testable without git.

## Acceptance Criteria

- [ ] AC-1 Spec: a stamp in which only non-golden inputs changed keeps the previous `engine` line.
- [ ] AC-2 Spec: a stamp in which a golden's blob changed records the running engine.
- [ ] AC-3 Spec: a first stamp (no previous file) records the running engine.

## Out of Scope

- A verification run that re-captures with no golden change keeps the old engine. The drift notice then stays visible, which is harmless.
- The notice's wording, `inputScopes`, and rendering goldens in CI.

## Avoided Traps

- **Failing the check on engine drift.** The engine tracks `dev` on its own schedule (#646); the notice must stay a notice.
- **"Restore the line by hand" as a rule.** It was missed once in two attempts.

## Related

#646 (the ruling), #278 (per-file stamp format), PR #850, PR #851.

Live latest-open + closed sweep: checked the latest 20 open issues, plus closed searches ("visual baseline stamp engine") and an org-wide search ("stamp-visual-baselines engine"). I read #278's and #646's Fix, Out of Scope and Avoided Traps; neither excludes this, and #646 is the ruling it follows. Swept at 2026-10-31T18:4xZ. A2A sweep (last 30, all read states): no claim on this scope. Memory Core sweep and own-assignment sweep: no equivalent.

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Retrieval Hint: "visual baseline stamp engine line captured against running engine input-only stamp"

Origin Session ID: 94dc6bfe-e91a-4ab9-9c0e-8cf1e2a2cf1e


## Timeline

- 2026-10-31T18:41:07Z @neo-opus-ada added the `bug` label
- 2026-10-31T18:41:07Z @neo-opus-ada added the `good first issue` label
- 2026-10-31T18:41:07Z @neo-opus-ada added the `ai` label
- 2026-10-31T18:41:08Z @neo-opus-ada added the `testing` label
- 2026-10-31T18:53:23Z @neo-opus-ada cross-referenced by PR #851
- 2026-10-31T18:53:40Z @neo-gemini-maryam assigned to @neo-gemini-maryam
- 2026-10-31T18:55:57Z @neo-gemini-maryam cross-referenced by PR #853
- 2026-10-31T19:06:55Z @tobiu referenced in commit `d9f2ceb` - "fix(build): keep visual-stamp engine line unless a golden changed (#852) (#853)"
- 2026-10-31T19:06:56Z @tobiu closed this issue

