---
id: 56
title: 'Nothing couples merged substrate to a published version, and this is the second ticket it has produced'
state: OPEN
labels:
  - enhancement
  - agent-os
assignees: []
createdAt: '2026-09-07T00:14:46Z'
updatedAt: '2026-09-07T00:16:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/56'
author: neo-opus-grace
commentsCount: 1
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
---
# Nothing couples merged substrate to a published version, and this is the second ticket it has produced

`Serves:` **none** — a live substrate defect, not a lane under an open epic. Stated rather than claimed.

## Context

Nothing in this repository couples merged substrate to a published version. `.github/workflows/` holds `reusable-pr-baseline.yml` and `skill-corpus.yml`; no step anywhere references `npm publish` or `NPM_TOKEN`. Release is a manual act with no trigger attached to it, and a manual step with no trigger is a state that must be manually cleared.

That is @tobiu's own retirement rationale for `not-code-ready`, which #52 already turns on:

> **A state that must be manually cleared will not be cleared.**

## This is the second manifestation, not the first

The same missing coupling has already produced one closed ticket, from the opposite direction:

| ticket | direction | symptom |
|---|---|---|
| #27 (COMPLETED) | pin → registry | the baseline pinned a `SKILLS_VERSION` that did not exist on npm |
| #44 (open) | merges → registry | twelve merged commits sit under an unbumped `0.1.3`, so no seat has them |

#27 was fixed by correcting the pin. The mechanism that produced both was not addressed, and it produced the second one four weeks later. A third is available in either direction at any time.

## Measured cost of the current instance

Recorded on #44: six shipped workflow references drift by **90 lines**, and `scripts/check-workflow-concurrency.mjs` — named in the `files` allowlist — exists on `dev` and on no installed seat. Five maintainers were reading `pull-request-workflow.md` and `ticket-create-workflow.md` 28 and 16 lines behind the merged text, with no signal that they were.

## The Problem

Two gaps, and the second is what makes the first stay fixed:

1. **No publish trigger.** A merged version bump does nothing. Someone must remember, from outside the repository.
2. **No bump enforcement.** A PR may change `.agents/skills` and ship no version change, and nothing objects. #41 was mine and did exactly this: it added a consumer-facing script to the `files` allowlist without a bump, and CI was green.

Fixing only (1) leaves the corpus able to drift again the moment the next PR forgets. Fixing only (2) turns every substrate PR into a manual publish reminder.

## The Fix

- A `publish` workflow triggered by a merged change to `package.json`'s `version`, so **the bump is the release** and no one has to remember a second act.
- A baseline CI arm that reds any PR whose diff touches `.agents/skills` or the `files` allowlist without changing `version`. It belongs in `reusable-pr-baseline.yml` alongside the guards already shared there, so consumer repositories inherit it rather than re-implementing it — the shape #25, #29 and #18 already established.

## Blocked on operator authority

Both halves need an `NPM_TOKEN` repository secret, which only @tobiu can create — an agent must never handle a registry credential.

**No implementation PR will be opened before that secret exists.** An unwired publish workflow is worse than the gap it claims to close: it reads as automation, runs on every merge, fails or silently no-ops, and the next person to look concludes publishing is handled.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| publish trigger | this ticket | merged `version` change publishes | manual publish, as today | repo README release section | no `npm publish` in any workflow |
| bump-enforcement arm | `reusable-pr-baseline.yml` precedent (#25, #29, #18) | `.agents/skills` diff without a bump is red | none — new gate | baseline job list | #41 merged green with no bump |
| `NPM_TOKEN` | @tobiu only | exists as a repository secret | ticket stays blocked | — | absent from all workflows |

## Acceptance Criteria

- **AC-1** A merge to `dev` that changes `version` publishes that version to npm; a merge that does not change it publishes nothing.
- **AC-2** A PR touching `.agents/skills` or the `files` allowlist without a `version` change fails CI, naming the file that forced the bump.
- **AC-3** The arm is contributed through the shared baseline, so a consumer repository inherits it without copying a job.
- **AC-4** `npm view neo-agent-skills version` equals `origin/dev`'s `package.json` `version` after any merge that changes it.

## Decision Record impact

`none`.

Filed by @neo-opus-grace (Claude Opus 5). Follow-up to #44 and PR #55, promised in both rather than smuggled into a one-line version bump.


## Timeline

- 2026-09-07T00:14:48Z @neo-opus-grace added the `enhancement` label
- 2026-09-07T00:14:48Z @neo-opus-grace added the `agent-os` label
- 2026-09-07T00:14:58Z @neo-opus-grace cross-referenced by #44
- 2026-09-07T00:16:24Z @neo-opus-grace cross-referenced by PR #55
### @neo-opus-grace - 2026-09-07T00:16:45Z

**Correction: this is a prerequisite of PR #55, not a follow-up to it.**

The repo's own guard settled it. `test-reusable-pr-baseline.mjs` asserts `SKILLS_VERSION` equals `package.json`'s version at all three install sites, so a bump cannot merge without moving the three pins in `reusable-pr-baseline.yml` with it. That means merging a bump puts `neo-agent-skills@<unpublished>` into three `npm install` lines that every consumer repository calls — **verbatim #27**, already closed COMPLETED once.

So merge-now-publish-later is not an available ordering. Either the publish happens from the branch before the merge, or this ticket lands first and the two stop being separable. That moves this from "nice to have, prevents future drift" to "the thing that makes a version bump safe at all".

🖖 @neo-opus-grace


- 2026-09-08T07:51:41Z @neo-opus-grace cross-referenced by #18465
- 2026-09-08T07:53:11Z @neo-opus-grace cross-referenced by PR #18466
- 2026-09-08T15:25:19Z @neo-opus-grace cross-referenced by PR #18485
- 2026-09-16T08:58:36Z @neo-opus-vega cross-referenced by #80
- 2026-09-16T09:21:02Z @neo-opus-vega cross-referenced by #81
- 2026-09-16T10:32:28Z @neo-opus-ada cross-referenced by PR #82
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

