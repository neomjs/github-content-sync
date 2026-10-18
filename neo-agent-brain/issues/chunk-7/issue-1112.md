---
id: 1112
title: provisioningTemplates.spec reads the engine's seat templates from the Brain tree
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-18T05:29:01Z'
updatedAt: '2026-10-18T14:41:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1112'
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
closedAt: '2026-10-18T14:41:45Z'
---
# provisioningTemplates.spec reads the engine's seat templates from the Brain tree

## Context

This is the next cluster of dev's Brain Unit base reds (the #201 census, comment 6245894428): `test/playwright/unit/ai/services/fleet/provisioningTemplates.spec.mjs`, 4 reds. All four fail with `ENOENT` on `.codex/config.template.toml` or `.claude/claude_desktop_config.example.json` under the Brain checkout.

## The Problem

The spec guards the seat templates Fleet provisions from: one credential slot, no literal credential, the Codex template's remote form, and the Claude Desktop bridge. It also checks each template's port against the Brain's Compose ingress. Both templates live in neomjs/neo, and Fleet reads them from the provisioned repository (`prepareManagedAgentWorkspace.mjs:1302`). The spec's `readRepoFile` resolves them from the Brain's own root, where they never were, so the guard has been dark since the split.

## The Architectural Reality

- The installed engine ships both templates (`node_modules/neo.mjs/.codex/config.template.toml`, `node_modules/neo.mjs/.claude/claude_desktop_config.example.json`). `neo.mjs` declares no `exports`, so `import.meta.resolve('neo.mjs/…')` reaches them. #1108 uses the same idiom.
- The Compose file (`deploy/cloud/docker-compose.local-agent-os.yml`) is the Brain's, so the port check compares the engine's templates with the Brain's ingress. That cross-repo comparison is the point of the guard.

## The Fix

Add a `readEngineFile` for the two templates. Keep `readRepoFile` for Compose.

Decision Record impact: none

## Acceptance Criteria

- [ ] **AC-1:** the 4 arms run and pass at head, and `provisioningTemplates.spec.mjs` passes whole.
- [ ] **AC-2:** the templates are read from the installed engine, and Compose is still read from the Brain.

## Out of Scope

- The templates' content, which the passing guard now covers.
- The other #201 clusters.

## Related

- #201: the burn-down.
- #1108 and #1110: sibling slices.

Live latest-open sweep: the latest 20 open Brain issues, checked at 2026-10-18T05:28:44Z; no equivalent found. `gh search` for the spec name found #201 only; #324 (closed) was a different spec.
A2A claim sweep: the last 6 messages; no claim.
Own-assignment sweep: #201.
Structure map: N/A; the change is test-only, in the spec's own file.

Origin Session ID: e2e1377f-66da-4e20-bb44-143ad59bd85b
Retrieval Hint: "provisioningTemplates spec ENOENT .codex/config.template.toml Brain checkout engine templates"

## Timeline

- 2026-10-18T05:29:02Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-18T05:29:03Z @neo-opus-ada added the `bug` label
- 2026-10-18T05:29:04Z @neo-opus-ada added the `ai` label
- 2026-10-18T05:29:04Z @neo-opus-ada added the `testing` label
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
- 2026-10-18T14:41:45Z @tobiu referenced in commit `36a04ef` - "fix(test): the seat-template guard reads the engine's templates (#1112) (#1113)

The spec read .codex/config.template.toml and .claude/claude_desktop_config.example.json from the Brain root, which never held them. readEngineFile reads them from the installed engine; Compose stays the Brain's, so the cross-repo port check runs again."
- 2026-10-18T14:41:46Z @tobiu closed this issue

