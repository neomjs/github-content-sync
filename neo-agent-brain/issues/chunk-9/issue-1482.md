---
id: 1482
title: 'npm run ai:audit-discussion-lifecycle does not exist'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-11-05T12:52:44Z'
updatedAt: '2026-11-05T13:33:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1482'
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
closedAt: '2026-11-05T13:33:37Z'
---
# npm run ai:audit-discussion-lifecycle does not exist

## Context

Two skill pages tell agents to run the Discussion lifecycle guard as `npm run ai:audit-discussion-lifecycle`: `neo-agent-skills` `.agents/skills/ideation-sandbox/audits/discussion-lifecycle-closure.md:18` and `references/ideation-sandbox-workflow.md:178`. Brain's `package.json` has no such script, so an agent following either page gets `Missing script`.

Grace measured the gap on #70 (2026-09-06). It still held on 2026-11-05 at `dev` `b282d7eb`. My premise read of #70 for #1438 (#70, 2026-11-05) split it out: the script is a bug in its own right, and it belongs before #70's scheduled workflow, so the workflow calls the same command the skills name.

## The Problem

The audit itself works. Run by its path against the synced corpus, `node ai/scripts/diagnostics/audit-discussion-lifecycle.mjs --discussions-dir <content-sync>/neo/discussions --json` scanned 181 Discussions on 2026-11-05 and found:
- 3 `graduated-open`, which fail the run;
- 10 `resolved-only-review` and 88 `stale-open`, both advisory.

Only the documented entry point is missing, so the guard reaches only someone who already knows the file path.

## The Architectural Reality

- The script is `ai/scripts/diagnostics/audit-discussion-lifecycle.mjs`. It resolves the corpus from `--discussions-dir` or the `issueSync.discussionsDir` config leaf, and refuses with a clear message when neither is set.
- Its sibling `ai:check-retired-primitives` (`package.json:62`) is the precedent: `node ./ai/scripts/diagnostics/<script>.mjs`.

## The Fix

One `package.json` entry: `"ai:audit-discussion-lifecycle": "node ./ai/scripts/diagnostics/audit-discussion-lifecycle.mjs"`. Arguments pass through `npm run … --`.

## Acceptance Criteria

- [ ] AC-1: `npm run ai:audit-discussion-lifecycle -- --discussions-dir <corpus> --json` runs the audit and exits as the script does: non-zero on `graduated-open`.
- [ ] AC-2: Without a configured corpus, the npm command shows the script's own refusal, not `Missing script`.

## Out of Scope

#70's scheduled workflow, standing alarm and graduation-time check. The skill pages, which already name this command.

## Related

#70 (the alarm this command feeds) · #1438 (the premise read that split it out)

Live latest-open + closed sweep: the latest 20 open Brain issues, plus searches (`ai:audit-discussion-lifecycle`, "audit discussion lifecycle npm script", "discussion lifecycle audit"), at 2026-11-05T12:52Z. The only hit is #70, whose body tracks the workflow, not the script. No A2A claim on it in the last hour. Own open assignments: no same surface.

Origin Session ID: 57180f73-f0b4-4ae5-bac8-095fd4813333

Retrieval Hint: "ai:audit-discussion-lifecycle npm script missing skills name"

Authored by Vega (Opus 5.5, Claude Code)

## Timeline

- 2026-11-05T12:52:44Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-05T12:52:46Z @neo-opus-vega added the `bug` label
- 2026-11-05T12:52:47Z @neo-opus-vega added the `ai` label
- 2026-11-05T12:52:47Z @neo-opus-vega added the `agent-os` label
- 2026-11-05T12:53:46Z @neo-opus-vega cross-referenced by PR #1483
- 2026-11-05T13:33:37Z @tobiu referenced in commit `ba1cb87` - "fix(scripts): npm run ai:audit-discussion-lifecycle runs the lifecycle audit (#1482) (#1483)"
- 2026-11-05T13:33:38Z @tobiu closed this issue
- 2026-11-05T13:51:32Z @neo-opus-vega cross-referenced by #277

