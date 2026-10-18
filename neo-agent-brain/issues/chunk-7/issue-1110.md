---
id: 1110
title: initTier1ConfigMigration.spec's fixtures import the engine from the Brain tree
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T05:26:32Z'
updatedAt: '2026-10-18T05:26:32Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1110'
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
---
# initTier1ConfigMigration.spec's fixtures import the engine from the Brain tree

## Context

dev's Brain Unit base carries 48 red tests (the #201 census, comment 6245894428). #1108 / PR #1109 cleared 13 of them. The next cluster is `test/playwright/unit/ai/initTier1ConfigMigration.spec.mjs`, with 6 reds: the overlay-conversion arms and the snapshot-cascade arm.

## The Problem

The spec writes disposable Tier-1 roots to a temp directory. The config files it generates import `${repoRoot}/src/Neo.mjs` and `${repoRoot}/src/core/_export.mjs`, which was the engine's own layout before the split. The Brain has no `src/`, so the first import fails with `Cannot find module '…/neo-agent-brain/src/Neo.mjs'`. The converter then refuses or warns where the arms expect `migrate`.

The production code under test is right: `migrateConfigOverlay.mjs:457-458` imports the engine by the bare `neo.mjs/src/…` specifier. Only the fixtures carry the stale path.

## The Architectural Reality

- Ten fixture lines (five Neo / core pairs) build the stale path. The fixtures' `${repoRoot}/ai/ConfigProvider.mjs` imports are the Brain's own and stay as they are.
- A fixture lives in a temp root, so a bare `neo.mjs` specifier cannot resolve there. It needs the installed engine's URL: `import.meta.resolve('neo.mjs/src/Neo.mjs')`, the idiom #1108 used.

## The Fix

Resolve `neoUrl` and `coreUrl` once beside `repoRoot`, and let the generated imports use them.

Decision Record impact: none

## Acceptance Criteria

- [ ] **AC-1:** the 6 arms run and pass at head, and `initTier1ConfigMigration.spec.mjs` passes whole.
- [ ] **AC-2:** no fixture imports an engine module from the Brain checkout's tree.

## Out of Scope

- The converter itself, which is correct.
- The other #201 clusters, which get one ticket each.

## Related

- #201: the burn-down.
- #1108: the sibling slice.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T05:26:10Z; no equivalent found. `gh search` for the spec name found only #201.
A2A claim sweep: the last 8 messages, all read-states; no claim on this spec.
Own-assignment sweep: #201, which this ticket serves.
Structure map: N/A; the change is test-only, in the spec's own file.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "initTier1ConfigMigration fixtures repoRoot/src/Neo.mjs engine layout after the Brain split"

## Timeline

- 2026-10-18T05:26:33Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T05:26:34Z @neo-opus-ada added the `bug` label
- 2026-10-18T05:26:34Z @neo-opus-ada added the `ai` label
- 2026-10-18T05:26:34Z @neo-opus-ada added the `testing` label
- 2026-10-18T05:27:11Z @neo-opus-ada cross-referenced by PR #1111
- 2026-10-18T05:29:03Z @neo-opus-ada cross-referenced by #1112
- 2026-10-18T05:29:29Z @neo-opus-ada cross-referenced by PR #1113
- 2026-10-18T05:32:05Z @neo-opus-ada cross-referenced by #201
- 2026-10-18T06:40:41Z @neo-opus-ada cross-referenced by #1117
- 2026-10-18T06:44:52Z @neo-opus-ada cross-referenced by PR #1118
- 2026-10-18T06:55:07Z @neo-opus-ada cross-referenced by #1119
- 2026-10-18T06:58:07Z @neo-opus-ada cross-referenced by PR #1120
- 2026-10-18T07:00:34Z @neo-opus-ada cross-referenced by #1121
- 2026-10-18T07:03:12Z @neo-opus-ada cross-referenced by PR #1122
- 2026-10-18T07:20:59Z @neo-opus-ada cross-referenced by #1125
- 2026-10-18T07:24:07Z @neo-opus-ada cross-referenced by PR #1126

