---
id: 978
title: Pass the host wake receiver’s state directory to Fleet
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T00:57:57Z'
updatedAt: '2026-11-15T00:57:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/978'
author: neo-gpt-emmy
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
# Pass the host wake receiver’s state directory to Fleet

## Context

Design authority: the operator wants native wakes available through normal FM Add and Start on an ordinary installation. The host receiver’s private state directory is a required input for [native seat provisioning](https://github.com/neomjs/neo-agent-brain/issues/1731). A Finder-launched app has no shell exports. Institution #400 already located the manifest and plane-reachable address from the deployment.

## The Problem

A receiver can keep the signed spool anywhere. `readReceiverLaunchAgent()` reads `--manifest` and `--port` but drops its `--state-dir`. The public manifest does not supply that directory. The previous canary required someone to pass it to an enrollment CLI. Inferring a sibling named `state` would select a different directory on valid installations.

## The Architectural Reality and Fix

Extend `harness/wakeReceiver.mjs`’s existing settlement and `wakeReceiverEnv()` to forward the selected receiver's absolute state directory through the Fleet child's existing environment composition in `main.mjs`. Use the existing deployment spelling `NEO_WAKE_RECEIVER_STATE_DIR`. Environment and plist are distinct receiver declarations: never fill a missing field from a different origin. Preserve the older manifest/base use for unrelated families when a native state directory is unavailable.

| Surface | Owner | Behavior | Evidence |
| --- | --- | --- | --- |
| Settled local receiver | Host launch env or parsed LaunchAgent ProgramArguments | Return its declared state directory, or `null` when unavailable/ambiguous | Fixture custom location, absent/invalid/duplicate flag, remote plane |
| Packaged Fleet child env | Shell wakeReceiverEnv | Pass or clear native state independently of legacy arming; preserve unpackaged inheritance | Child-boundary tests with conflicting ambient input |

## Acceptance Criteria

- [ ] Finder/local-plane path reads the real state directory from the same plist as manifest and port, including locations unrelated to the manifest directory.
- [ ] Explicit environment declaration uses only its own absolute state directory. Incomplete, relative, ambiguous or mismatched declarations never silently select an inherited or guessed native path. Missing native state preserves previously working legacy coordinates.
- [ ] Unpackaged behavior still inherits normally; capture each case through the existing child-env seam. No operator-specific path, secret, new UI switch or live profile change.

Decision Record impact: none, extends the #400 settlement contract. Native setup, receiver installation and trust belong to their existing owners. Related: neomjs/neo-agent-brain#1731 · neomjs/neo-agent-brain#571.
Live latest-open + closed sweep: latest 20 Institution issues, 58 open (red), checked closed “wake receiver” and state-directory search. #400 shipped two coordinates. This supplies the third to the same consumer. Recent A2A claim sweep found no collision. Implementation owner Emmy; not a newcomer leaf.
Origin Session ID: 1188c53b-f31f-4f66-a94b-fc696eb02810
Retrieval Hint: "Finder Fleet host wake receiver declared state directory launch agent native enrollment"

## Timeline

- 2026-11-15T00:57:57Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T00:57:59Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-15T00:57:59Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T00:57:59Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T01:04:43Z @neo-gpt-emmy cross-referenced by PR #979
- 2026-11-15T01:09:26Z @neo-gpt-emmy cross-referenced by #980
- 2026-11-15T01:15:18Z @neo-gpt-emmy cross-referenced by #1731

