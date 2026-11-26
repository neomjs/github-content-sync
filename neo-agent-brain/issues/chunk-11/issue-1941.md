---
id: 1941
title: Verify the carrier-specific Codex wake plist
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - build
  - agent-os
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-26T02:03:33Z'
updatedAt: '2026-11-26T03:11:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1941'
author: neo-gpt-sophie
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
blocking:
  - '[ ] 1054 Outer Fleet metadata changes the Codex wake hook address'
closedAt: '2026-11-26T03:11:40Z'
---
# Verify the carrier-specific Codex wake plist

Paired with neomjs/neo-agent-institution#1054. Sophie owns the paired implementation.

## Context

The installed v1 selector includes the outer `Contents/Info.plist` verbatim in an unsigned immutable Codex wake carrier. An isolated replay changed only the outer `ElectronAsarIntegrity.Resources/app.asar` value and changed the carrier address. The carrier does not include `app.asar`. [Reproduction and packaging owner](https://github.com/neomjs/neo-agent-institution/issues/1054)

## The Problem

The publisher and managed Start must compute the same digest over the actual carrier. A publisher-only transform would make the verified source descriptor disagree with the carrier's physical bytes. Existing v1 packages also need their original address.

## The Architectural Reality

Design authority: an immutable artifact's path is derived from its complete executable payload's canonical inventory. [ADR 0034 §2.5](https://github.com/neomjs/neo-agent-institution/blob/5720069e1e77892b11242cdb897642071dde4968/learn/agentos/decisions/0034-electron-shell-architecture.md#L497-L506)

`ai/services/fleet/codexWakeArtifact.mjs` owns manifest selection. The Institution publisher, Start verifier and retention reuse it through the runtime-generation export. The format change adds a reader and preserves v1.

## The Fix

Support a v2 declaration with an attested, sealed plist inside the organism. Project its physical inventory entry to the existing `Contents/Info.plist` output position and expose its fixed source to the publisher. Publish explicit format support so the packer and installer can reject an incompatible reader early.

Decision Record impact: aligned-with ADR 0034 §2.5.

## Acceptance Criteria

- [ ] v1 descriptors and digests remain compatible. The v2 descriptor derives the Info.plist row from the selected sealed payload and changes the address if any actual carrier input changes.
- [ ] v2 rejects a missing, linked or writable plist source. The projected inventory agrees with a physically published carrier and excludes the sidecar itself.
- [ ] Verify the paired package publisher and no-window runtime on Institution #1054. The disposable managed-seat approval and delivery checks are tracked there.

## Out of Scope

Signed carriers, user trust state, and unrelated wake protocol changes. Curated labels omitted because the reader and the package publisher ship as a paired maintainer change.

Live sweep: newest 20 of 155 open, open and closed wake-artifact searches, Brain #1803 and PR #1805 reviewed. No owner for the v2 plist descriptor. Latest 30 mailbox messages show no competing claim. Own-assignment and Memory Core sweeps found no parallel reader change. Structure map confirms existing `ai/services/fleet` owner.

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822
Retrieval Hint: “Codex wake carrier v2 normalized plist projected inventory and Institution #1054”

## Timeline

- 2026-11-26T02:03:34Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-26T02:03:36Z @neo-gpt-sophie added the `bug` label
- 2026-11-26T02:03:36Z @neo-gpt-sophie added the `ai` label
- 2026-11-26T02:03:37Z @neo-gpt-sophie added the `regression` label
- 2026-11-26T02:03:37Z @neo-gpt-sophie added the `build` label
- 2026-11-26T02:03:37Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-26T02:12:36Z @neo-gpt-sophie cross-referenced by PR #1942
- 2026-11-26T02:13:49Z @neo-gpt-sophie marked this issue as blocking #1054
- 2026-11-26T02:14:04Z @neo-gpt-sophie cross-referenced by #1054
- 2026-11-26T02:15:49Z @neo-gpt-sophie cross-referenced by PR #1055
- 2026-11-26T03:11:40Z @tobiu referenced in commit `63bbaa1` - "feat(fleet): verify carrier-specific wake plists (#1941) (#1942)"
- 2026-11-26T03:11:41Z @tobiu closed this issue

