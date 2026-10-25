---
id: 814
title: 'Pin the engine to a neo dev commit that carries #20107'
state: CLOSED
labels:
  - bug
  - ai
  - dependencies
assignees:
  - neo-opus-ada
createdAt: '2026-10-25T17:21:01Z'
updatedAt: '2026-10-25T18:13:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/814'
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
closedAt: '2026-10-25T18:12:10Z'
---
# Pin the engine to a neo dev commit that carries #20107

### Context

neo #20106 ("A list writes a changed record into its stale pre-sort slot") is a defect in the installed Fleet Manager's roster. The operator's roster showed an empty card slot covered by another card on 2026-10-24 and again on 2026-10-25. neo #20107 fixed it and merged at 2026-10-25T16:45Z as `ea457faa6d`.

The next walk candidate is built from this repository's lock. The lock resolves `neo.mjs` to `a97cb2d5f8` (set by #810), which predates the fix. Walk candidate 2 (engine `0c1a373918`) predates it too. Tuesday's row-2 walk changes seat states, which is the defect's trigger, so the walker has the defect named in the candidate-2 packet. A candidate 3 can carry the fix only once this lock moves.

### The Problem

`package.json` declares `"neo.mjs": "github:neomjs/neo#dev"`, and the lock fixes the commit (`package-lock.json`, `node_modules/neo.mjs` → `resolved`). A packaged build (`harness` → `npm run dist`) installs from the lock. So the Fleet Manager carries the lock's engine commit until the lock moves.

### The Architectural Reality

Delta census for `a97cb2d5f8..0eb2229acf`, which is neo `dev` at 2026-10-25T17:1xZ. Read row by row (`git diff --stat`):
- `src/list/Base.mjs` (+4/−3) and `src/list/Buffered.mjs` (+5/−5): #20107's fix. `onStoreRecordChange` now takes the record's current index from the store, not the payload's pre-sort `index`.
- `0eb2229acf` (#20108) is test-only.

The delta changes no `package.json`, no `resources/scss` and no engine version (13.3.1 stays). The Institution consumer is the roster List (`apps/agentos/view/fleet/roster/`), whose controller re-sorts the store on `recordChange`; that is the trigger #20106 names. The Brain pin does not move.

### The Fix

A lock-only move, the same shape as #807. Run `npm install neo.mjs` so that `#dev` resolves to a commit containing `ea457faa6d`. Then walk the pin checklist:
1. Install for real: remove `node_modules/neo.mjs` first, then confirm the fix in the installed `src/list/Base.mjs`.
2. Run the unit suites lock-faithful (no `NEO_AGENTOS_RUNTIME_ROOT`), as CI runs them.
3. Rebuild the themes (`npm run build-themes -- -n -e dev -t all`), then run the visual suite at both engines. Re-stamp only if all of it is green.
4. Check for spy-host doubles. None are expected, since the delta adds no reactive config.

### Acceptance Criteria

- [x] `package-lock.json` resolves `node_modules/neo.mjs` to a neo `dev` commit that contains `ea457faa6d`, and the installed `src/list/Base.mjs` carries the fix. *Merged in #815 (`ee6ab02b6b`, 2026-10-25T18:12Z). The lock resolves `0eb2229acf`; the installed list files are byte-identical to it.*
- [x] Institution CI is green at the new lock, and the visual suite passes at the old engine and the new one, or each difference is named. *Merged in #815 (`ee6ab02b6b`, 2026-10-25T18:12Z). The one difference, `observatory-pane-widened`, fails at both engines with identical pixel deltas (#815's Test Evidence; defect note 3bbda9fa).*

### Out of Scope

- Cutting candidate 3 itself: the build, rehearsal, packet and install. That follows this merge and #813's, on the operator's yes. If neither merges by Mon 2026-10-26 12:00 CET, candidate 2 stands for Tuesday with the defect named.
- Any Brain pin move.

### Related

neo #20106 / #20107 (the fix) · #806 / #807 (the previous pin move, same shape) · #810 (the current lock) · #516 (the walk) · #813.

Decision Record impact: none.

Live latest-open sweep: the latest 20 open Institution issues and every open PR at 2026-10-25T17:20Z, no equivalent; an exact search for `20107` found nothing. A2A in-flight claim sweep: the latest 30 messages, all read states, no claim on an engine pin (Grace's merge note names the pin as follow-up). Memory Core sweep: no prior decision. Own-assignment sweep: #812, #516 and #424, none overlapping. Structure map: N/A (lock-only).

unowned-rationale: none, self-assigned. Not `good first issue`, because it is time-bound to Tuesday's walk.

Origin Session ID: 531ac06b-0bfa-40a3-8577-2cff0699ff08

Retrieval Hint: "engine pin lock neo.mjs roster stale pre-sort slot empty card #20107 walk candidate 3"

## Timeline

- 2026-10-25T17:21:02Z @neo-opus-ada added the `bug` label
- 2026-10-25T17:21:03Z @neo-opus-ada added the `ai` label
- 2026-10-25T17:21:03Z @neo-opus-ada added the `dependencies` label
- 2026-10-25T17:21:07Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-25T17:45:05Z @neo-opus-ada cross-referenced by PR #815
- 2026-10-25T18:12:10Z @tobiu closed this issue
- 2026-10-25T18:12:10Z @tobiu referenced in commit `ee6ab02` - "fix(deps): engine pin → dev@0eb2229acf, which re-renders a changed list record at its current index (#814) (#815)

Carries neo #20107, the fix for the FM roster's stale pre-sort slot (neo #20106). Lock-only; the delta census is in the ticket."

