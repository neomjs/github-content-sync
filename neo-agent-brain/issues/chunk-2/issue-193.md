---
id: 193
title: Establish canonical source and domain ownership
state: OPEN
labels:
  - epic
  - ai
  - refactoring
  - architecture
  - performance
  - agent-os
  - tech-debt
assignees: []
createdAt: '2026-08-27T15:01:38Z'
updatedAt: '2026-08-28T22:20:26Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/193'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 212
subIssues:
  - '[x] 71 Nothing distinguishes a deliberate graph handle from an accidental one'
  - '[x] 215 Extract REM digestion into the Evolution context'
  - '[x] 217 Expose one client-safe Fleet contract from Brain'
subIssuesCompleted: 3
subIssuesTotal: 3
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 201 Run the retained Brain unit suite in CI'
---
# Establish canonical source and domain ownership

## Problem scope

Brain behavior is organized primarily by technical buckets such as `services`, `daemons`, `mcp`, `scripts`, and `graph`. Large coordinators consequently combine storage, policy, transport, scheduling, recovery, and provider effects, while generic shared folders conceal who owns a capability.

The previous prescription compounded that problem by assigning durable-intelligence domains to Cloud and treating Dream as a domain. Host and Cloud are executable profiles, not business ownership. REM digestion is an Evolution use case; the Orchestrator schedules it but does not own its implementation.

## Intended solution shape

Establish one canonical `src/**` authority organized around cohesive Brain domains and application use cases. Move retained behavior by responsibility, deleting obsolete code and tests within each slice. The legacy `ai/**` production root disappears as those slices land; it is not copied wholesale into a prettier tree.

Keep composition and transport at the edges. Shared executables and services remain shared when multiple profiles or domains genuinely use the same contract. Host and Cloud profiles choose concrete effectful adapters before construction; they do not own mirrored source.

Prefer direct imports, small factories, and explicit parameters. Extract a module only when it creates a real ownership boundary or a second consumer—not merely to reduce a line count.

## Why this is an Epic

Memory, knowledge, Evolution, orchestration, collaboration, graph persistence, and their current technical buckets cannot converge in one reviewable PR. Each child must deliver a working, smaller domain slice rather than a taxonomy document.

## Out of scope

- embedding admission, coordinated by #23;
- executable profiles and deployment artifacts, owned by #213;
- test discovery/CI policy, owned by #194;
- a dependency-injection container, service locator, or domain registry.

## Avoided traps

- `cloud/src`, `host/src`, or any mirrored production tree;
- declaring every current folder a domain;
- one technical-layer stack repeated inside every domain;
- moving every current class unchanged;
- wrapper extraction whose only outcome is a shorter file.

## Related

Parent: #212. #215 is the first corrective Evolution slice. #71 remains a storage-ownership input whose assumptions must be revalidated inside the relevant domain.


## Timeline

- 2026-08-27T15:01:40Z @neo-gpt-emmy added the `epic` label
- 2026-08-27T15:01:41Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:01:41Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:01:41Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:01:41Z @neo-gpt-emmy added the `performance` label
- 2026-08-27T15:01:41Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:01:42Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy cross-referenced by #199
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T22:17:05Z @neo-gpt-emmy cross-referenced by #215
- 2026-08-28T22:20:26Z @neo-gpt-emmy changed title from **Refactor Dream and durable intelligence by domain** to **Establish canonical source and domain ownership**
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #191
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #198
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T22:29:58Z @neo-opus-vega cross-referenced by #71
- 2026-08-28T22:31:06Z @neo-gpt-emmy cross-referenced by #126
- 2026-08-28T22:33:55Z @neo-gpt-emmy cross-referenced by #217
- 2026-08-28T22:36:00Z @neo-gpt-emmy cross-referenced by #139
- 2026-08-28T23:05:14Z @neo-gpt-emmy cross-referenced by #201
- 2026-08-28T23:38:43Z @neo-gpt-emmy cross-referenced by #200
- 2026-08-29T00:01:02Z @neo-gpt cross-referenced by PR #220
- 2026-08-29T03:54:34Z @neo-gpt-emmy cross-referenced by PR #228
- 2026-08-30T23:54:44Z @neo-opus-ada cross-referenced by #89
- 2026-08-31T03:14:12Z @neo-opus-grace cross-referenced by #271
- 2026-09-05T00:54:59Z @neo-gpt-emmy cross-referenced by #18349
- 2026-09-05T01:19:57Z @neo-gpt-emmy cross-referenced by PR #326

