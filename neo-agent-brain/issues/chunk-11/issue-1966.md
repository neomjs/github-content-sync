---
id: 1966
title: Expose the Engine gesture trace through the public Neural Link action
state: OPEN
labels:
  - enhancement
  - ai
assignees: []
createdAt: '2026-11-27T21:22:56Z'
updatedAt: '2026-11-27T21:24:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1966'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 22400
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 22407 Replay bounded gesture contact traces in the Engine’s owning window'
blocking: []
---
# Expose the Engine gesture trace through the public Neural Link action

## Context

Peers need to replay a contact sequence against an app and verify a component outcome. The public `simulate_event` tool successfully replayed eight contact transitions on the stock Helix through Main’s JSON Touch materializer, but callers had to expand every native-event list and construct target/window coordinates themselves. [Public MCP, fixture and browser comparison](https://github.com/neomjs/neo/discussions/22259#discussioncomment-19961766)

## The Architectural Reality

The [G2 decision](https://github.com/neomjs/neo/issues/22400) gives Engine ownership of recognition, coordinates, provider execution and cleanup. Brain already forwards interaction requests through [InteractionService](https://github.com/neomjs/neo-agent-brain/blob/00762332/ai/services/neural-link/InteractionService.mjs#L72) and publishes write-locked tool schemas. neomjs/neo#22407 owns the RPC and neomjs/neo#22408 owns fixture/browser conformance. The schema must match the reviewed Engine contract in ADR 0045 (neomjs/neo#22401).

## The Fix

Expose the bounded trace and named presets through a discoverable public gesture action, using the existing interaction service and write-ownership boundary. Forward the requested session/window, target, trace or preset, and provider requirement to the owning implementation. Publish the capability matrix and handbook. Keep event construction and gesture thresholds on the Engine side.

## Acceptance Criteria

- [ ] AC-1 On a real connected app, a peer can request the same staggered-lift, cancellation and restart trace used by the Playwright fixture. Read the observed provider, active/changed contacts, trust and termination, then independently verify the app’s result.
- [ ] AC-2 Invalid traces and missing required provider capabilities return a bounded refusal. A demand for browser-owned input cannot silently fall back to successful synthetic dispatch. Competing writers and a disconnected session preserve the existing write-ownership guarantees.
- [ ] AC-3 Publish the tool’s input/output schema, discovery text and capability limits with the other Neural Link actions. Cover forwarding through the public front door and both positive and refusal controls.

Engine v14.1 integration parent: neomjs/neo#22400. Peers can self-select. The Engine RPC is the runtime prerequisite.
 
Origin Session ID: c3328164-c307-4c22-b385-606a978d46f0
Retrieval Hint: public drive_gesture Neural Link engine browser providers contact trace write lock

## Timeline

- 2026-11-27T21:22:57Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-27T21:22:58Z @neo-gpt-emmy added the `ai` label
- 2026-11-27T21:23:18Z @neo-gpt-emmy added parent issue #22400
- 2026-11-27T21:23:19Z @neo-gpt-emmy marked this issue as being blocked by #22407
- 2026-11-27T21:44:16Z @neo-gpt-euclid cross-referenced by #22400
- 2026-11-27T22:53:46Z @neo-gpt-euclid cross-referenced by PR #22433

