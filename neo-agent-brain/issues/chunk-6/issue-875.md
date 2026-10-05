---
id: 875
title: 'Runtime identity facts come from the plane''s graph, not identityRoots'
state: OPEN
labels:
  - epic
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-vega
createdAt: '2026-10-05T10:32:47Z'
updatedAt: '2026-10-05T11:17:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/875'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues:
  - '[x] 874 Start fleet reads a seat''s participation from its identity node, not the seed'
  - '[x] 879 Wake eligibility reads a seat''s participation from its identity node'
  - '[ ] 880 The heartbeat and issue focus read participation from the identity node'
subIssuesCompleted: 2
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Runtime identity facts come from the plane's graph, not identityRoots

Terminal predicate: no Agent OS service, daemon or MCP tool depends on `ai/graph/identityRoots.mjs` at runtime. Every identity fact they act on comes from the plane's identity graph, and the facts an operator owns about their own agents are set in Fleet Manager's Agent Detail, so an operator without our roster runs the same Agent OS.

## Problem scope

**The ruling.** The operator, in session on 2026-10-05, relayed on #874 ([5992651676](https://github.com/neomjs/neo-agent-brain/issues/874#issuecomment-5992651676)): "we can update identity graph infos, however: agent os and MCP tools should never use them. this only works for OUR own team, but can not work for other operators who simply do not even have that file for them." Clarified later the same day: "of course peers can read it (this is why we have the file), but other FM operators do not have it for THEIR agent teams. this is why i recommended that code should avoid using it. however: we could enhance what is inside the FM agent details view => like the benched toggle."

**What it touches.** Ada's census on that comment, a grep of `dev` that counts modules, not behaviors: 16 runtime modules import the roots directly, and `agentFamilyResolution.mjs` re-exports them. They cover:
- participation gates;
- family and alias resolution;
- trust tiers;
- mailbox addressing;
- memory, session and summary services;
- ingestion;
- concept-touch measurement;
- Fleet open-work sourcing;
- the Memory Core server.

For our team the file and the graph mostly agree, after a reseed. For another operator the file holds someone else's team, so every one of those readers answers about seats that are not theirs, and about none of their own.

**Already owned elsewhere:** the family and alias read paths move under #112, which sits in the identity-state epic #114. That ticket's carve-out (trust tiers, wake routes and mailbox addresses "stay identity-level") predates the ruling, so those readers belong here.

**Why an epic.** Each reader family is its own one-PR leaf with its own regression surface: alias resolution and wake routing must behave identically before and after. No single PR could review them all.

## Intended solution shape

- **The roots stay our team's file and seed.** Peers read them. The explicit projection (`ai/scripts/setup/seedAgentIdentities.mjs`) writes them to the graph. No runtime path depends on them.
- **Every runtime reader takes the graph's answer.** Plane-side services read identity nodes in-process. The Fleet DTO takes what the plane already sends it, the `who_is_online` presence snapshot, as #874 found.
- **An unanswered read stays visible.** It is named as unanswered, never silently replaced by a local file or a hidden default.
- **What an operator owns gets a control.** A fact the roots record for our team, but that another operator must set for their own agents, becomes a row in Fleet Manager's Agent Detail, written to the graph. Bench and unbench comes first (#28).
- **A guard, last:** a lint that refuses a runtime import of the roots, so the predicate stays true after the leaves land.

## Out of scope

- The roots' content: it stays the seed for this deployment.
- The participation write path (#28).
- The era and identity-state schema (#114, #151).

## Avoided traps

- **Reading the roots as a fallback when the graph cannot answer.** For another operator that fallback is a wrong answer, not a degraded one.
- **One large migration PR.** The census spans services whose regressions are unrelated.

Structure map: run 2026-10-05 for `ai/graph` and `ai/scripts/setup`. The roots and the seeder keep their folders; no file moves.

Origin Session ID: 79265a5a-6888-4d34-94ee-0d933cbacff1

Retrieval Hint: `query_raw_memories("Agent OS must never read identityRoots at runtime, graph is the identity source")`


## Timeline

- 2026-10-05T10:32:48Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-05T10:32:49Z @neo-opus-vega added the `epic` label
- 2026-10-05T10:32:49Z @neo-opus-vega added the `ai` label
- 2026-10-05T10:32:49Z @neo-opus-vega added the `architecture` label
- 2026-10-05T10:32:49Z @neo-opus-vega added the `agent-os` label
- 2026-10-05T10:32:58Z @neo-opus-vega added sub-issue #874
- 2026-10-05T10:33:33Z @neo-opus-vega cross-referenced by #874
- 2026-10-05T10:36:13Z @neo-opus-vega cross-referenced by #876
- 2026-10-05T11:11:52Z @neo-opus-vega cross-referenced by PR #878
- 2026-10-05T11:48:05Z @neo-opus-vega cross-referenced by #879
- 2026-10-05T11:48:07Z @neo-opus-vega cross-referenced by #880
- 2026-10-05T11:48:14Z @neo-opus-vega added sub-issue #879
- 2026-10-05T11:48:16Z @neo-opus-vega added sub-issue #880
- 2026-10-05T12:09:00Z @neo-opus-vega cross-referenced by PR #882
- 2026-10-05T12:16:06Z @neo-opus-vega cross-referenced by #31
- 2026-10-05T12:18:16Z @neo-opus-vega cross-referenced by #883
- 2026-10-05T12:47:55Z @neo-opus-vega cross-referenced by #885
- 2026-10-05T13:04:52Z @neo-opus-vega cross-referenced by #568
- 2026-10-05T13:30:56Z @neo-gpt-sophie cross-referenced by PR #884
- 2026-10-05T14:16:05Z @neo-gpt-sophie cross-referenced by PR #886
- 2026-10-05T15:15:58Z @neo-gpt cross-referenced by PR #890

