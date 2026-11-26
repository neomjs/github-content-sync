---
id: 1054
title: Outer Fleet metadata changes the Codex wake hook address
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
  - build
  - regression
assignees:
  - neo-gpt-sophie
createdAt: '2026-11-26T01:00:32Z'
updatedAt: '2026-11-26T08:27:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/1054'
author: neo-gpt-sophie
commentsCount: 1
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 1941 Verify the carrier-specific Codex wake plist'
blocking: []
closedAt: '2026-11-26T08:21:48Z'
---
# Outer Fleet metadata changes the Codex wake hook address

## Context

A managed Codex seat stopped receiving wakes after Fleet changed the carrier paths in its approved submit hook. The receiver had accepted the wakes, and Codex reported the hook as Modified. Replacing only the two carrier addresses in the current definition reproduced the exact earlier configuration hash. The hook entrypoint had identical bytes. [Prior installed recurrence](https://github.com/neomjs/neo-agent-brain/issues/1735#issuecomment-7048785408)

## The Problem

The two retained unsigned carriers differed in four entries. Three wake modules also changed, so that pair alone cannot demonstrate a package-only invalidation. In `Contents/Info.plist`, the only changed value was `ElectronAsarIntegrity.Resources/app.asar`. That archive belongs to the outer Fleet app and is absent from the carrier. With the other carrier entries held fixed, substituting only that plist row changed the digest returned by the installed selector.

## The Architectural Reality

Design authority: “The packaged macOS Codex submit hook may be shared by two Starts when both of their sealed generations declare the same complete wake execution payload.” [ADR 0034 §2.5](https://github.com/neomjs/neo-agent-institution/blob/5720069e1e77892b11242cdb897642071dde4968/learn/agentos/decisions/0034-electron-shell-architecture.md#L497-L506)

The package declares the wake dependencies. Brain selects `Contents/Info.plist` in full, and Fleet publishes and verifies the immutable selected tree. The resulting digest enters both paths in the vendor-reviewed hook command. [Packer](https://github.com/neomjs/neo-agent-institution/blob/5720069e1e77892b11242cdb897642071dde4968/harness/packCodexWake.mjs) · [Selector](https://github.com/neomjs/neo-agent-brain/blob/191aaaa7c98db873914720a00202d97fcd4b48cf/ai/services/fleet/codexWakeArtifact.mjs#L32-L65)

## The Fix

Make the unsigned carrier's sealed plist reflect the carrier's own files and required launch metadata. Keep publication, Start verification and retention on one canonical inventory. Test a real Electron-as-Node hook run from the resulting carrier as well as vendor approval continuity. Keep the existing signed release path under #1007.

Decision Record impact: aligned-with ADR 0034 §2.5.

## Acceptance Criteria
- [ ] Two unsigned package fixtures differing only in outer `app.asar` integrity metadata derive the same physical carrier address. The full outer package inventories remain distinct.
- [ ] A changed executable input selects a new carrier address. Publication and Start refuse an unverifiable carrier.
- [ ] The real Electron-as-Node wake entrypoint runs from the published carrier and produces the expected scratch presence marker.

## Post-Merge Validation
- [ ] [L4-deferred — installed test] Build and install two unsigned packages with different outer app hashes and identical wake inputs on a disposable managed seat. Verify the second Start preserves the genuine Codex hook definition and approval, then verify one native wake with its receiver and same-chat receipt. Residual-Owner: neomjs/neo-agent-brain#571. [Native delivery trial](https://github.com/neomjs/neo-agent-brain/issues/1735)

## Out of Scope

The signed carrier in #1007 and the operator-visible detection gap in neomjs/neo-agent-brain#1373 have separate owners. The observed changes in shared wake modules require their own dependency audit before any scope reduction.

## Avoided Traps

Hash the bytes actually published and executed. Keep the outer app's integrity metadata and Codex's trust decision under their current owners.

## Related

Verifier prerequisite neomjs/neo-agent-brain#1941. Prior independent carrier work #1005 and #1012. Installed acceptance neomjs/neo-agent-brain#571 and neomjs/neo-agent-brain#1735.

Live latest-open and noun sweep: 20 latest open of 58 checked, plus open and closed “wake hook” search on 2026-11-26. #1007 is signed-only. Closed #1005 and #1012 do not isolate this plist input.
A2A: latest 30 across read states, no overlapping claim. Memory Core: four results, verified against the existing issue and source. Own assignments: three, none cover carrier packaging. Structure map: Brain `ai/services/fleet` and `ai/daemons/wake`; Institution producer and publisher remain in `harness/`.

Retrieval Hint: “Codex wake artifact ElectronAsarIntegrity plist package-only hook reapproval”

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822

## Timeline

- 2026-11-26T01:00:34Z @neo-gpt-sophie added the `bug` label
- 2026-11-26T01:00:34Z @neo-gpt-sophie added the `agent-os` label
- 2026-11-26T01:00:35Z @neo-gpt-sophie added the `ai` label
- 2026-11-26T01:00:35Z @neo-gpt-sophie added the `build` label
- 2026-11-26T01:00:35Z @neo-gpt-sophie added the `regression` label
- 2026-11-26T01:00:52Z @neo-gpt-sophie cross-referenced by #1373
- 2026-11-26T02:01:28Z @neo-gpt-sophie assigned to @neo-gpt-sophie
- 2026-11-26T02:03:34Z @neo-gpt-sophie cross-referenced by #1941
- 2026-11-26T02:12:36Z @neo-gpt-sophie cross-referenced by PR #1942
- 2026-11-26T02:13:49Z @neo-gpt-sophie marked this issue as being blocked by #1941
- 2026-11-26T02:15:49Z @neo-gpt-sophie cross-referenced by PR #1055
- 2026-11-26T03:18:05Z @neo-gpt-sophie referenced in commit `be38266` - "build(deps): pin merged wake carrier verifier (#1054)"
- 2026-11-26T08:21:48Z @tobiu referenced in commit `0b3ba02` - "fix(harness): keep outer ASAR metadata out of wake carriers (#1054) (#1055)

* fix(harness): keep outer ASAR metadata out of wake carriers (#1054)

* build(deps): pin merged wake carrier verifier (#1054)"
- 2026-11-26T08:21:50Z @tobiu closed this issue
### @neo-gpt-sophie - 2026-11-26T08:27:01Z

### Post-merge verification

Institution `dev` includes merge commit `0b3ba0254cf28eca573be24f2523d49ec2955a72`. All five changed files match the reviewed `be382668` head byte-for-byte. Both hosted Institution CI lanes passed on that head. [Merged PR](https://github.com/neomjs/neo-agent-institution/pull/1055) · [CI run](https://github.com/neomjs/neo-agent-institution/actions/runs/45748030290)

The package installed on the validation host still reports product `aa975704` and wake declaration v1 as of 08:23 UTC. The installed acceptance remains open on [Brain #571](https://github.com/neomjs/neo-agent-brain/issues/571). Its witness is approval continuity across two v2 packages with identical wake inputs and different outer archives, followed by one native wake received in the intended Codex chat.

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822


