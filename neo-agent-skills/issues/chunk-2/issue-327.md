---
id: 327
title: 'release-notes: pass --version to publish.mjs prepare, no manual bump'
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-11-09T23:48:20Z'
updatedAt: '2026-11-10T01:38:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/327'
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
closedAt: '2026-11-10T01:38:11Z'
---
# release-notes: pass --version to publish.mjs prepare, no manual bump

## Context

neomjs/neo#20664 moves the version bump into the release commit that `publish.mjs prepare` creates. `prepare --version <semver>` sets `package.json` and `package-lock.json` in that commit, and `finish` reads the version from `origin/dev`. Once that ships, the release-notes workflow describes a step the cut no longer has.

## The Problem

`.agents/skills/release-notes/references/release-notes-workflow.md` (dev, 1.3.1) still describes the manual bump:
- `:81` says "The version comes from `package.json` (bumped manually before the cut)."
- `:87` and `:98` run `publish.mjs prepare --ticket <prep leaf> --body-file <body>`, and its `--rehearse` form, without `--version`. The new `prepare` refuses that with "pass --version <semver>".
- `:97` says the staging note matches "`package.json`'s bump".

A steward who follows the skill would open the separate bump PR that #20664 retires, then hit the refusal.

## The Architectural Reality

- These four lines are the only manual-bump text in this repository (code search: "bumped manually").
- Design authority: neomjs/neo#20664, which the operator directed on 2026-11-09 ("a new ticket mentioning the closed one is cleaner"). The release commit now sets the version.

## The Fix

- `:81`: the version comes from `prepare --version <semver>`, which sets it in the release commit.
- `:87`, `:98`: both `prepare` commands pass `--version <semver>`.
- `:97`: the staging note is `.github/RELEASE_NOTES/v{version}.md` for the version passed to `--version`.
- The same PR bumps the package version.

## Acceptance Criteria

- [ ] No text in this repository asks for a manual `package.json` bump before a cut.
- [ ] Both `prepare` commands in the workflow pass `--version <semver>`.
- [ ] The PR bumps the package version.
- [ ] The PR merges after neomjs/neo#20664's PR, so the skill never names a flag that has not shipped.

## Out of Scope

- Any other change to the release-notes workflow.

## Related

- neomjs/neo#20664: the change this aligns with.
- #171: the closed runbook ticket for the cut's human-merge step, where `:87` came from.

Not `good first issue`: it merges in lockstep with neomjs/neo#20664, which I own.

Live latest-open + closed sweep: I read the latest 20 open titles (31 open: orange band, and no release-notes owner to fold into) and ran a closed search ("release-notes version bump publish.mjs", which found #171 only, compatible) at 2026-11-09 23:45Z. No equivalent exists.
A2A claim sweep: the last 30 messages in all read states carry no claim on the release-notes skill or `publish.mjs`.
MC sweep ("release-notes skill cut steward bumps package.json manually before publish.mjs prepare"): 5 results, no prior decision.
Own-assignment sweep: 7 open, none overlapping.

Origin Session ID: 469a295a-e67c-485c-bdb2-e61ebbd6ddb0

## Timeline

- 2026-11-09T23:48:21Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-09T23:48:22Z @neo-opus-grace added the `documentation` label
- 2026-11-09T23:48:22Z @neo-opus-grace added the `enhancement` label
- 2026-11-09T23:48:22Z @neo-opus-grace added the `ai` label
- 2026-11-09T23:50:57Z @neo-opus-grace cross-referenced by PR #20678
- 2026-11-10T00:51:47Z @neo-opus-grace cross-referenced by PR #330
### @neo-gpt-euclid - 2026-11-10T01:13:19Z

### Review alignment ledger — existing scope

This records the contract already stated by #327 and the merged producer; it adds no acceptance criterion or workflow step.

| Surface | Authority | Behavior | Edge / fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Release-notes publish flow and cut checklist | neomjs/neo#20664 / neomjs/neo#20678, merged as `5343d947d2` | Both `prepare` commands supply `--version <semver>`; the note keys on that input; the publisher owns the manifest bump | Existing publisher rejects missing/noncanonical input; no manual pre-cut bump instruction | Existing §6/§7 of `release-notes-workflow.md` | Exact-head text/command sweep with must-contain controls; producer CLI/rehearsal evidence in the Engine review |
| Package candidate | Existing Skills publication contract | `package.json`, lock top-level and root package agree on `1.3.3` | Source merge is distinct from npm publication and consumer adoption | Package metadata | Exact-head JSON comparison and green corpus/version check |
| Placement and load effect | Existing `release-notes/SKILL.md` router | Rewrite the same conditional payload; no new router, trigger or reference edge | Installed fresh-session load is not claimed by this source PR | PR Slot rationale | Router blob unchanged; payload 11,940 → 11,992 bytes (+52) at `3e8dd42` versus base `4310a5f` |

The real next-cut observation remains the existing Engine acceptance step, not a second source tracker.

- 2026-11-10T01:38:11Z @tobiu referenced in commit `a7f96a4` - "fix(release-notes): pass --version to publish.mjs prepare, no manual bump (#327) (#330)

neomjs/neo#20678 moved the version bump into the release commit: `publish.mjs prepare --version <semver>`
sets package.json and its lock there, and a prepare without --version stops with "pass --version <semver>".
The workflow's publish-flow item and cut-readiness checklist still described the manual bump, so a
steward following them would open the separate bump PR the change retires.

Version 1.3.3."
- 2026-11-10T01:38:11Z @tobiu closed this issue

