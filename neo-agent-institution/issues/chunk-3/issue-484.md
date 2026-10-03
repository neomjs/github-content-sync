---
id: 484
title: 'The Institution pins Brain fb40366: the Fleet boots without a GitHub token'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-03T08:45:52Z'
updatedAt: '2026-10-03T09:15:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/484'
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
closedAt: '2026-10-03T09:15:37Z'
---
# The Institution pins Brain fb40366: the Fleet boots without a GitHub token

## Context

Institution #12's installed cut at Brain `804356b` failed: the packaged Fleet refused to boot without a GitHub token. Emmy rolled it back. The fix merged as Brain #794 (`fb40366`, 2026-10-03 08:44Z). The re-cut, and with it #571's team enrollment, needs an Institution package that pins it.

## The Fix

Move the Brain pin `804356b` → `fb403664f110fe0957941a92ba6b8e835191263e` in its three places: `package.json`, `package-lock.json`, and the Brain checkout's `ref:` in `.github/workflows/ci.yml`.

**Delta census** (`git diff --stat 804356b fb40366 -- src package.json`): **no rows**, so no Body-safe contract changed. ~~so no Institution consumer moves~~: corrected after the isolated unit run. The setup broker (`harness/setupBroker.mjs`) runs the Brain's setup modules directly, not through `src/`, so #790's contract reaches it. An interrupted effect that cannot settle now halts with `{ok: false, reason}` naming why, instead of answering `ok` over a `reconcile-required` row. Its witness in `harness/setupBroker.spec.mjs` moves with this pin; the card's branch for that reply is #475's. The range is 5 Brain PRs, 22 files:
- #785 (served-plane observer reads `/mc/mcp`)
- #789 (ADR 0041 §3 sentence)
- #790 (an interrupted host-file effect settles by its own observation; merged at head `7c1928c`)
- #791 (the wake delivery reader holds terminal records)
- #794 (the Fleet boots without a GitHub token)

## Acceptance Criteria

- [ ] AC-1: The three pin places name `fb40366`, and a fresh install carries `readGithubToken` in `node_modules/neo-agent-brain/ai/services/ingestion/githubActions.mjs`.
- [ ] AC-2: The isolated CI job and the Brain-bound unit contract pass at the new pin.

## Post-Merge Validation

- [ ] The packaged Fleet built from this pin boots from Finder with no GitHub token in its environment (#12's re-cut, by its install owner).

## Out of Scope

- The re-cut itself (#12).

## Related

#12, #571 (the team's enrollment waits on the re-cut). Brain #793 / #794.

Live latest-open sweep: latest 20 open Institution issues and every open PR at 2026-10-03T08:45:18Z; no pin ticket or PR in flight. A2A claim sweep: no pin claim; I told Emmy at 07:17Z I would carry it. Structure map: N/A (manifest and workflow only).

Origin Session ID: 258e3158-432b-49ad-9cbe-b1568e69e7d1


## Timeline

- 2026-10-03T08:45:53Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-03T08:45:54Z @neo-opus-ada added the `enhancement` label
- 2026-10-03T08:45:54Z @neo-opus-ada added the `agent-os` label
- 2026-10-03T08:45:54Z @neo-opus-ada added the `ai` label
- 2026-10-03T08:48:55Z @neo-opus-vega cross-referenced by #486
- 2026-10-03T08:51:03Z @neo-opus-ada referenced in commit `8747529` - "feat(deps): pin Brain fb40366 — the Fleet boots without a GitHub token (#484)

Moves the Brain pin 804356b to fb40366 in package.json, the lockfile and the CI checkout. The range
brings #794 (the Fleet boots without a GitHub token, which #12's failed cut needs), #785, #789, #790
and #791; it changes no Body-safe contract. The setup broker runs the Brain's setup modules
directly, so #790's contract reaches its witness: an interrupted effect that cannot settle now halts
with the reason it is unsettled instead of answering ok over a reconcile-required row."
- 2026-10-03T08:51:22Z @neo-opus-ada cross-referenced by PR #488
- 2026-10-03T09:15:37Z @tobiu referenced in commit `e1a9dbe` - "feat(deps): pin Brain fb40366 — the Fleet boots without a GitHub token (#484) (#488)

Moves the Brain pin 804356b to fb40366 in package.json, the lockfile and the CI checkout. The range
brings #794 (the Fleet boots without a GitHub token, which #12's failed cut needs), #785, #789, #790
and #791; it changes no Body-safe contract. The setup broker runs the Brain's setup modules
directly, so #790's contract reaches its witness: an interrupted effect that cannot settle now halts
with the reason it is unsettled instead of answering ok over a reconcile-required row."
- 2026-10-03T09:15:37Z @tobiu closed this issue
- 2026-10-03T09:53:03Z @neo-opus-vega cross-referenced by #495

