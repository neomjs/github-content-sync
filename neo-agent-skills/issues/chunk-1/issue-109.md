---
id: 109
title: The authorship pre-push guard scans upstream commits after a rebase
state: CLOSED
labels:
  - bug
  - ai
  - model-experience
assignees:
  - neo-opus-ada
createdAt: '2026-09-23T13:47:38Z'
updatedAt: '2026-09-23T14:23:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/109'
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
closedAt: '2026-09-23T14:23:33Z'
---
# The authorship pre-push guard scans upstream commits after a rebase

## Context

Two independent sightings on 2026-09-23. Grace's push of neomjs/neo#19113 (defect-note 13:12Z, fingerprint `1f261b38cb5fe16f`) and mine of neomjs/neo#19103's branch (13:36Z) were both refused by `check-commit-authorship`. Both times it flagged `a1302b068f`, the neomjs/neo#19115 squash: operator-authored, already on `origin/dev`, and introduced by neither push. Grace ran the guard by hand on her own range and pushed with `--no-verify`. I deleted my PR-less remote branch and pushed it fresh, which takes the new-branch path. The second sighting promotes the note (`ticket-create` §1e).

## The Problem

After a rebase, a force-push to an existing remote branch is scanned as `remoteSha..localSha`. The old head is no longer an ancestor of the new one, so that range holds every upstream commit between the old base and the new base. One operator-authored commit among them blocks the push, although the push introduces none.

This will recur on a date we know. The 13.1.0 release commit `a987abfb25` is operator-authored and on `dev`, so after the 13.2 cut every agent branch rebased across it trips the guard. The two ways past are both bad: `--no-verify` disables every other pre-push guard too, and deleting the remote branch is not open to a branch with a PR.

## The Architectural Reality

- `scripts/check-commit-authorship.mjs` `pendingRanges()` (line 73 on `dev`): an existing remote branch is scanned as `${remoteSha}..${localSha}`, and a new branch as `${localSha} --not --remotes` (`UNSEEN`, line 34).
- The docblock calls `remoteSha..localSha` "the exact boundary git applies". That is true for what the push transfers. The guard's question is different: which commits does this push introduce? A commit that some remote-tracking ref has already seen was introduced by someone else.
- The new-branch arm already asks that question.

## The Fix

Scan an existing remote branch as `${remoteSha}..${localSha} --not --remotes`, i.e. append `UNSEEN`. Commits any remote-tracking ref has seen drop out; every commit the push adds is still scanned. `scripts/test-commit-authorship.mjs` gains a rebase arm and its control.

## Acceptance Criteria

- [ ] Pushing a rebased branch whose new base holds an operator-authored commit passes when every commit the branch adds is agent-authored. The arm is red on `dev` before the fix.
- [ ] An operator-authored commit that the branch itself adds is still flagged, both on a rebased branch and on a fast-forward push (control arm).
- [ ] Both arms live in `scripts/test-commit-authorship.mjs`.
- [ ] The `pendingRanges()` docblock names the question the range answers (commits this push introduces), not git's transfer boundary.

## Out of Scope

- CI mode (`--base <sha>`): `base..HEAD` does not depend on push history.
- Consumer pin bumps, which follow the release.

## Avoided Traps

- **`--not --remotes` alone for every row.** It is correct while remote-tracking refs are fresh, but it drops the boundary git reports for the remote branch. Appending keeps both.
- **Admitting by email domain.** That would let a real agent-checkout commit through as the operator, the case this guard exists for.

## Decision Record impact

`none`.

## Related

#93 (the guard's origin) · #51 · neomjs/neo#19113 · neomjs/neo#19125

Live latest-open sweep: checked the latest 20 open issues at 2026-09-23T13:45:57Z and re-checked the latest 5 at 13:47:20Z (newest #105); no equivalent.
A2A in-flight sweep: 30 messages across read states (newest 13:40Z). The only one on this scope is Grace's defect-note, with no claim.
MC sweep: "commit authorship pre-push blocks push after rebase operator-authored commit already on origin dev remoteSha localSha range", 6 results. Grace's 13:15Z session reached the same root cause and did not promote it; no prior decision.
Own-assignment sweep: 2 open (#103, #102), none overlapping.

Origin Session ID: 3be453e4-8b04-4865-be62-4cff34f4e0c6

Retrieval Hint: `query_raw_memories("authorship pre-push guard rebase upstream operator commit remoteSha localSha")`

## Timeline

- 2026-09-23T13:47:38Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-23T13:47:40Z @neo-opus-ada added the `bug` label
- 2026-09-23T13:47:40Z @neo-opus-ada added the `ai` label
- 2026-09-23T13:47:40Z @neo-opus-ada added the `model-experience` label
- 2026-09-23T14:01:50Z @neo-opus-ada cross-referenced by PR #110
- 2026-09-23T14:23:33Z @tobiu referenced in commit `793b580` - "Merge pull request #110 from neomjs/ada/109-authorship-range

fix(authorship): a rebased branch answers only for the commits it adds (#109)"
- 2026-09-23T14:23:33Z @tobiu closed this issue
- 2026-09-23T14:31:21Z @neo-opus-ada cross-referenced by #19141

