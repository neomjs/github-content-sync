---
id: 1121
title: 'PackageBoundary.spec: a version-pinned setup-node lookup, and a root script that duplicates the Cloud package''s'
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T07:00:33Z'
updatedAt: '2026-10-18T14:43:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1121'
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
closedAt: '2026-10-18T14:43:35Z'
---
# PackageBoundary.spec: a version-pinned setup-node lookup, and a root script that duplicates the Cloud package's

## Context

This is a #201 cluster: two reds in `test/playwright/unit/deploy/PackageBoundary.spec.mjs`.

## The Problem

- **`integration installs the independent package and then tests the exact checkout`.** The arm finds the setup step by `step.uses === 'actions/setup-node@v6'`. Since the actions-group bump `be9ec2a1` (09-15), `brain-integration.yml` uses `@v7`. The lookup returns `undefined`, and the arm throws on `.with`. The arm tests the cache paths, not the action's version.
- **`the Cloud package owns its manifest, lock, dependencies, and commands independently`.** `ai:check-backup-integrity` is defined in both manifests. #197 moved it into the Cloud package and added this no-overlap rule. #277 then re-added it to the root, to satisfy `backup.spec`'s canary, which reads the root `package.json`. #277's premise was that no manifest defined the script, but the Cloud package did.

## The Architectural Reality

- `RestorationRunbook.md`'s other commands are `ai:restore`, `ai:reseed`, `ai:backup`, `ai:check-chroma-integrity`, `ai:sync-kb` and `ai:server`, and every one is defined only in `cloud/package.json`. The runbook is a Cloud-package document, and the root entry is the outlier.
- No other consumer of the root script exists. In the Brain, neo, neo-agent-skills, neo-stage and the Institution, the name appears only in the runbook and the two specs.

## The Fix

1. Find the setup step by its action name, at any version.
2. Remove `ai:check-backup-integrity` from the root `package.json`.
3. Point `backup.spec`'s canary (`a named npm script invokes the timeline tool`) at `cloud/package.json`, the manifest the runbook's commands run in.

## Acceptance Criteria

- [ ] **AC-1:** `PackageBoundary.spec.mjs` passes at head. On dev, 2 of its 5 tests fail.
- [ ] **AC-2:** `backup.spec.mjs`'s canary passes against `cloud/package.json`, and the root manifest no longer defines the script.

## Out of Scope

- The other #201 clusters.

## Related

- #201: the burn-down.
- #197: the package split.
- #277: the root re-add.
- #1108, #1110, #1112, #1117, #1119: sibling slices.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T07:00:13Z; no equivalent found. Open-PR sweep: only dependabot #1063 touches `package.json`, at lines 112+.
A2A claim sweep: the latest 20 inbox messages; no claim.
Own-assignment sweep: #201, #1117, #1119.
Structure map: N/A; this changes two specs and one manifest line, and adds no `.mjs`.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "PackageBoundary setup-node v7 lookup; ai:check-backup-integrity root/cloud overlap; backup.spec canary reads cloud manifest"

## Timeline

- 2026-10-18T07:00:33Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T07:00:34Z @neo-opus-ada added the `bug` label
- 2026-10-18T07:00:35Z @neo-opus-ada added the `ai` label
- 2026-10-18T07:00:35Z @neo-opus-ada added the `testing` label
- 2026-10-18T07:03:12Z @neo-opus-ada cross-referenced by PR #1122
- 2026-10-18T07:14:13Z @neo-opus-ada cross-referenced by #1123
- 2026-10-18T07:20:59Z @neo-opus-ada cross-referenced by #1125
- 2026-10-18T07:24:28Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T07:27:25Z @neo-opus-ada cross-referenced by #1127
- 2026-10-18T07:33:14Z @neo-opus-ada cross-referenced by #1129
- 2026-10-18T07:58:38Z @neo-opus-ada cross-referenced by #1131
- 2026-10-18T14:43:35Z @tobiu referenced in commit `3ea511e` - "fix(test): the package boundary finds setup-node at any version, and the Cloud package alone owns the backup canary (#1121) (#1122)

The integration arm matched actions/setup-node@v6 exactly, which the actions-group bump moved to @v7. The root re-add of ai:check-backup-integrity duplicated the Cloud package's entry, where every other restoration-runbook command lives, so the root entry goes and backup.spec's canary reads cloud/package.json."
- 2026-10-18T14:43:35Z @tobiu closed this issue

