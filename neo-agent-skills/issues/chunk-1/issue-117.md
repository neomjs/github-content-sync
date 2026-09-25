---
id: 117
title: 'A dependabot pull request can never pass the close-target check, so every version bump reds PR body'
state: CLOSED
labels:
  - bug
  - contributor-experience
  - ai
  - github_actions
assignees:
  - neo-opus-grace
createdAt: '2026-09-24T20:39:37Z'
updatedAt: '2026-09-25T10:25:12Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/117'
author: neo-opus-grace
commentsCount: 0
parentIssue: 14
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-25T10:25:12Z'
---
# A dependabot pull request can never pass the close-target check, so every version bump reds PR body

## Context

#103 made the close target repository policy for every pull request: `Validate the close target` runs for any author, draft included, from 0.1.16. A dependabot body never carries `Resolves #N`, so every dependabot PR now reds `PR body`:
- **Measured:** 0.1.17's `neo-agent-skills-pr-body --close-target-only` exits 1 on the bodies of neomjs/neo#19048 and neomjs/neo-agent-institution#188, two merged dependabot PRs.
- **Why neo hasn't seen it yet:** its last dependabot PRs ran before neo's pin reached 0.1.16. In run 35700214894, both body steps were skipped.
- **Who else hits it:** #114 AC-4 expects dependabot to move `@vX.Y.Z` on each release. That PR, and every npm bump, reds `PR body` in all three consumers.

@tobiu merges bot PRs without a ticket: neomjs/neo#19048, #19025, #18815 and #18756, and neomjs/neo-agent-institution#188. @neo-gpt found this on neomjs/neo-agent-institution#190.

## The Fix

Judge the close target for every author except a GitHub App: `github.event.pull_request.user.type != 'Bot'`. For a Bot author, a step logs that the close target was not judged and why, so its green is not the vacuous pass #103 removed. Human and agent PRs are judged exactly as now. A missing author type is not `Bot`, so it is judged (fail closed).

**Load effect:** none. The reusable workflow is not loaded into any agent's context.

## Contract Ledger (T3)

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `reusable-pr-baseline.yml` `pr-body` job, `Validate the close target` | #103 (every PR resolves one ticket); the operator merges bot PRs without one | Runs for every author whose type is not `Bot`, draft included | A missing or unknown author type is judged | The job's comment | `test-reusable-pr-baseline` required entry and mutations |
| The same job, Bot author | This ticket | A step states that a `Bot` author's close target is not judged, naming the login | None; the step only reports | The job's comment | The same suite |

## Acceptance Criteria

- [ ] **AC-1:** `Validate the close target` carries `user.type != 'Bot'`, and a Bot-only step logs the unjudged close target with the login.
- [ ] **AC-2:** `test-reusable-pr-baseline` requires the condition. Two mutations fail: dropping it (a dependabot PR is judged) and widening it to exempt a `User`.
- [ ] **AC-3:** The job comment states the Bot scope beside #103's every-author rule.

## Related

- **Parent epic:** #14 (unify PR governance).
- **Rule this scopes:** #103.
- **Owner of the live dependabot PR:** #114 (AC-4).

Sweeps at 20:40Z:
- **Latest open:** the latest 20 open issues, none on bot authors or close targets.
- **Search:** `dependabot`, `bot`, `close target` and `Resolves bot` find only the closed #103, #74 and #88.
- **A2A:** no claim.

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T20:39:38Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-24T20:39:39Z @neo-opus-grace added the `bug` label
- 2026-09-24T20:39:39Z @neo-opus-grace added the `contributor-experience` label
- 2026-09-24T20:39:39Z @neo-opus-grace added the `ai` label
- 2026-09-24T20:39:40Z @neo-opus-grace added the `github_actions` label
- 2026-09-24T20:39:48Z @neo-opus-grace added parent issue #14
- 2026-09-24T20:40:09Z @neo-opus-grace cross-referenced by #114
- 2026-09-24T20:45:00Z @neo-opus-grace cross-referenced by PR #190
- 2026-09-24T20:45:08Z @neo-opus-grace cross-referenced by PR #40
- 2026-09-24T20:48:21Z @neo-opus-grace cross-referenced by PR #118
- 2026-09-25T10:25:12Z @tobiu referenced in commit `905675a` - "Merge pull request #118 from neomjs/grace/117-bot-close-target

fix(baseline): a bot's pull request is reported, not judged, for its close target (#117)"
- 2026-09-25T10:25:12Z @tobiu closed this issue
- 2026-09-25T10:32:00Z @neo-opus-grace cross-referenced by #19206
- 2026-09-25T11:02:37Z @neo-opus-vega cross-referenced by PR #19207

