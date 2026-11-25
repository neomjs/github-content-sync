---
id: 471
title: The premise catalog lists VDom update-path files the engine guides now own
state: CLOSED
labels:
  - enhancement
  - ai
  - refactoring
assignees:
  - neo-opus-ada
createdAt: '2026-11-25T17:32:38Z'
updatedAt: '2026-11-25T19:19:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/471'
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
blockedBy:
  - '[x] 467 Plugin bundles ship dead relative links to unbundled skills'
blocking: []
closedAt: '2026-11-25T19:19:28Z'
---
# The premise catalog lists VDom update-path files the engine guides now own

## Context

#465 (Resolves #463) added the "Engine update path without guides and ADR" row to `.agents/skills/pr-review/audits/cycle-1-premise-preflight.md`. Its note enumerates the update path's files and ends with a promise: "Until the guides land (neo#22189), every such diff fires the row; then the path list moves into them and this note shrinks to their link."

The guides have landed in `neomjs/neo` (`learn/guides/vdomengine/`: `Overview.md`, `AppWorker.md`, `VdomWorker.md`, `MainThread.md`). Checked on neo `dev` `e6418ff`, every path the note lists appears in them:
- `src/vdom/` and `src/worker/VDom.mjs` in `VdomWorker.md`
- `src/util/vdom/`, `src/manager/VDomUpdate.mjs` and the `needsVdomUpdate` cascade in `AppWorker.md`
- `src/main/DeltaUpdates.mjs` and `src/Main.mjs` with its queues in `MainThread.md`
- `src/mixin/VdomLifecycle.mjs` in all four

## The Problem

The note now keeps a second copy of a fact the guides own, and the two copies drift separately: a file added to the update path updates the guide, not this note. Its "Until the guides land" sentence is also stale.

## The Architectural Reality

- The audit is skill-loaded from `pr-review-guide.md` §0 through the premise catalog. It is not turn-loaded, so placement does not change.
- The row's trigger question in the catalog table stays as it is. Only the note under the table changes.
- Design authority: the note's own sentence, quoted above, and the operator's rule of 2026-11-25 that it cites (guides → ADR → code).

## The Fix

1. Replace the path enumeration and the stale "Until the guides land" sentence with one link to `learn/guides/vdomengine/`, the files those guides walk from collection to sync-back.
2. Bump the package to the next free version (2.9.2, since #469 holds 2.9.1) and regenerate the Claude plugin bundles (`generate-claude-plugin`, then `--check`).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `cycle-1-premise-preflight.md` "Engine update path" note | operator rule 2026-11-25 (neo#22178) · `learn/guides/vdomengine/` | names the update path by linking the guides instead of listing files | none: the row's question and Drop+Supersede default are unchanged | the audit itself | `npm run lint` byte budgets · plugin `--check` |

## Acceptance Criteria

- [ ] AC-1: the note names the update path by linking `learn/guides/vdomengine/`, with no file enumeration and no "Until the guides land" sentence.
- [ ] AC-2: the audit's loaded bytes shrink.
- [ ] AC-3: the version is bumped to the next free version, the plugin bundles are regenerated, and `generate-claude-plugin --check` and `npm run lint` pass.

## Out of Scope

- The row's trigger question and its Drop+Supersede default.
- Any change to the guides themselves.

## Decision Record impact

None.

## Related

#463 / #465 (the row) · neo#22178 (operator rule) · neo#22189 (the guides ticket)

Ownership: assigned to me. This is the post-merge step of my own #465. Curated contributor labels stay off because the work is owned and happens now.

Live latest-open + closed sweep: checked the latest 20 open plus open and closed searches for "premise catalog VDom update path" at 2026-11-25T17:30Z. Only #463, closed and the row's own ticket, came up; no equivalent found. 45 open, which is the orange band, and no owning open ticket carries this.
A2A sweep: the last 30 messages, all read states. No claim on this scope.
MC sweep: "premise catalog VDom update path note path list shrink to guides link cycle-1-premise-preflight", 5 results. No prior decision found.
Own-assignment sweep: 1 open (#298), not on this surface.

Origin Session ID: cc55a843-2d87-4222-b61d-8efb38ee837d

## Timeline

- 2026-11-25T17:32:38Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-25T17:32:40Z @neo-opus-ada added the `enhancement` label
- 2026-11-25T17:32:41Z @neo-opus-ada added the `ai` label
- 2026-11-25T17:32:41Z @neo-opus-ada added the `refactoring` label
- 2026-11-25T17:36:10Z @neo-opus-ada cross-referenced by PR #472
- 2026-11-25T18:26:46Z @neo-opus-ada marked this issue as being blocked by #467
- 2026-11-25T19:19:28Z @tobiu referenced in commit `d2d4d10` - "docs(pr-review): link the engine guides for update-path files (#471) (#472)

The premise catalog's "Engine update path" note listed the update path's
files and promised to shrink to a link once the engine guides landed.
They have landed, and they name every listed path, so the note now
links learn/guides/vdomengine instead of keeping a second copy that
drifts. The row's question is unchanged. Version 2.9.2, with the plugin
bundles regenerated."
- 2026-11-25T19:19:30Z @tobiu closed this issue

