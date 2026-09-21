---
id: 213
title: Establish executable profiles and declarative deployment
state: OPEN
labels:
  - epic
  - ai
  - refactoring
  - testing
  - architecture
  - build
  - agent-os
  - tech-debt
assignees: []
createdAt: '2026-08-28T22:14:38Z'
updatedAt: '2026-08-28T22:14:38Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/213'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 212
subIssues:
  - '[x] 214 Move runtime profiles out of deployment artifacts'
  - '[x] 198 Remove Engine projections after Brain source takes ownership'
  - '[x] 12 Receive Agent OS deployment and prove the Brain image'
  - '[x] 184 Align Brain''s Engine pin with post-split consumers'
  - '[ ] 216 The plan gate refuses on required leaves a service never reads'
subIssuesCompleted: 4
subIssuesTotal: 5
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# Establish executable profiles and declarative deployment

## Problem scope

The Brain currently conflates three different concerns: canonical source ownership, executable dependency selection, and deployment packaging. That confusion produced the architecture in #192 and the merged #197 shape, where Cloud package authority and executable JavaScript live under `deploy/**`, while the Host entrypoint still imports essentially the same eager service graph after applying a posture flag.

Host and Cloud need different effectful capabilities, credentials, dependencies, and runtime packaging. They do not need duplicated domains, services, daemons, or test trees.

## Intended solution

Establish Host and Cloud as explicit executable profiles over one domain-owned source tree.

The Host profile is the default developer and CI execution environment. The root manifest exposes only Host Edge commands that have a current owner and proven use. The Cloud profile owns a separate nested manifest and lockfile outside `deploy/**`, containing only Cloud-specific runtime dependencies and commands.

Composition roots select the allowed task catalog and concrete adapters before constructing shared executables. Factories and constructor parameters make those dependencies visible. No context may import another profile's overlay, concrete store singleton, or mutable global posture.

Deployment directories become declarative consumers of the profiles: Dockerfiles, Compose, Caddy, launchd, and static configuration. Tests stay Host-executed; integration tests create the containers they need.

## Why this is an Epic

Correcting the boundary touches manifests, composition roots, deployment artifacts, CI, local launch paths, and dependency closure. These must converge without turning deployment layout into source architecture.

## Out of scope

- splitting shared domains or executables into Host and Cloud copies;
- building a dependency-injection framework;
- moving every package dependency in one change;
- inventing a Cloud-only test runner.

## Traps to avoid

- no production or test `.mjs` under `deploy/**`;
- no `cloud/src` or `deploy/cloud/src`;
- no root scripts retained merely because they existed before the split;
- no runtime branch that imports both profiles and decides later;
- no package boundary inferred from a Docker build context alone.

This supersedes #192 and replaces the architectural direction embodied by closed #197.

## Timeline

- 2026-08-28T22:14:39Z @neo-gpt-emmy added the `epic` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `architecture` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `build` label
- 2026-08-28T22:14:40Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T22:14:41Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-28T22:16:16Z @neo-gpt-emmy cross-referenced by #214
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #191
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #193
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #198
- 2026-08-28T22:22:05Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T22:22:06Z @neo-gpt-emmy cross-referenced by #192
- 2026-08-28T22:22:13Z @neo-gpt-emmy cross-referenced by #197
- 2026-08-28T22:25:01Z @neo-opus-vega cross-referenced by #212
- 2026-08-28T22:31:05Z @neo-gpt-emmy cross-referenced by #91
- 2026-08-28T22:32:03Z @neo-gpt-emmy cross-referenced by #216
- 2026-08-28T22:48:58Z @tobiu cross-referenced by #184
- 2026-08-29T17:51:53Z @neo-gpt-emmy cross-referenced by PR #235
- 2026-08-29T19:47:50Z @neo-opus-vega cross-referenced by PR #236
- 2026-08-30T00:16:17Z @neo-fable-clio cross-referenced by #246
- 2026-08-30T17:45:56Z @neo-gpt-emmy cross-referenced by PR #255

