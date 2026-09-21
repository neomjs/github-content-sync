---
id: 214
title: Move runtime profiles out of deployment artifacts
state: CLOSED
labels:
  - bug
  - ai
  - refactoring
  - testing
  - architecture
  - build
  - agent-os
  - tech-debt
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-28T22:16:15Z'
updatedAt: '2026-08-29T19:48:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/214'
author: neo-gpt-emmy
commentsCount: 0
parentIssue: 213
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[x] 198 Remove Engine projections after Brain source takes ownership'
blocking:
  - '[ ] 216 The plan gate refuses on required leaves a service never reads'
closedAt: '2026-08-29T19:48:49Z'
---
# Move runtime profiles out of deployment artifacts

## Context

Closed #197 placed package/runtime authority and executable JavaScript under `deploy/cloud/**` and `deploy/host/**`. The accepted architecture in #212 and #213 reverses that direction: deployment is declarative, source is canonical, and Host/Cloud differences are executable profiles.

Current-`dev` evidence:

- `deploy/cloud/package.json` and `package-lock.json` own the nested Cloud package;
- `deploy/host/hostEdgeProfile.mjs` owns production Host profile logic;
- `deploy/cloud/mock-oidc-server.mjs` and `mock-openai-embedding-server.mjs` are test executables;
- the root manifest exposes 58 scripts and the Cloud manifest exposes 36, so retention must be justified by current consumers rather than inherited by default.

## The Problem

`deploy/**` currently mixes declarative deployment definitions with JavaScript source, test fixtures, package authority, and runtime command policy. That makes deployment layout part of source architecture and obscures which profile owns each executable and dependency.

The package placement also contradicts ADR 0040's settled topology: the repository root is the Host-Edge package and `cloud/` is the independently installed nested package. A package under `deploy/cloud/` turns Compose/Caddy placement into npm authority.

## The Architectural Reality

There is one canonical source authority. Host Edge and Container Cloud are executable profiles over shared domains, not mirrored source trees. Profile selection belongs at composition edges before shared executables are constructed; Docker, Compose, Caddy, and launchd consume those profiles but do not own executable JavaScript.

AiConfig remains the reactive Provider SSOT per ADR 0019. Moving the Host profile changes source custody, not the sanctioned mechanism: the entrypoint still supplies deployment inputs before importing the daemon graph, and consumers continue to read resolved leaves at use sites.

Pre-Flight (structural full): considered `deploy/host/`, `ai/daemons/orchestrator/`, and `src/composition/orchestrator/` for the Host profile. `deploy/host/` is rejected because deployment must be declarative; `ai/daemons/orchestrator/` would deepen the legacy technical bucket the source-root Epic is dissolving; `src/composition/orchestrator/` makes the profile an explicit composition-edge input beside the shared executable without creating `cloud/src` or a second domain tree. ADR 0040 requires a root Host package plus independent `cloud/` package with no workspaces; ADR 0019 requires deployment inputs before AiConfig use-site reads. Map maintenance is required because this establishes the canonical composition role. Ten more profiles remain grouped by executable instead of forming a generic scripts bucket; no new ADR is needed because #212/#213 and ADR 0040 already decide the trade-off; future extraction is easier because deployment artifacts consume a source-owned composition contract.

## The Fix

- Move the Cloud manifest and lockfile to top-level `cloud/`, preserving an independently installable package with no workspace or ancestor-hoist dependency.
- Move Host profile source to `src/composition/orchestrator/` and update the thin Host entrypoint to consume it before daemon import.
- Move the OIDC and embedding mock servers into test fixtures.
- Update Docker, Compose, launchd, CI, tests, and current operator guides to consume the new authorities.
- Audit both manifests against live consumers. Delete inherited aliases without a current CI, operator, package, or executable consumer; record retained-script evidence in the PR, not in a new repository ledger.
- Extend the package-boundary test so executable source and package authority under `deploy/**` fail mechanically.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Root `package.json` | ADR 0040 Host-Edge root contract | exposes only proven Host/CI/developer commands and dependencies | absent commands fail loud; no forwarding to Cloud | README + Host operation guide | retained-script consumer table and exact command probes |
| `cloud/package.json` + lockfile | ADR 0040 independent Cloud package | installs alone, owns Cloud-only commands/dependencies, and pins the Brain dependency | no workspace, ancestor hoist, or root fallback | README + Cloud tutorial/cookbook | disposable `npm ci`, package-root command probes, lockfile readback |
| `src/composition/orchestrator/hostEdgeProfile.mjs` | #212/#213 executable-profile boundary + ADR 0019 deployment-input rule | source-owned Host profile is applied before daemon graph import | contradictory explicit role refuses; ordinary explicit overrides remain authoritative | Host operation guide | focused Host profile specs and entrypoint import check |
| `deploy/host/**`, `deploy/cloud/**` | deployment consumers | declarative deployment/static configuration only | executable or package artifacts fail the boundary test | deployment guides | zero `.mjs`, `package.json`, lockfile, or `.npmrc` under `deploy/**` |
| test mock servers | Host-executed test ownership | fixtures live under `test/**` and are started only by Compose test/parity overlays | missing fixture fails the invoking test | test JSDoc | focused Compose rendering plus exact changed specs |

## Decision Record impact

Aligned-with ADR 0040 and ADR 0019. This corrects implementation placement; it does not amend either decision.

## Acceptance Criteria

- [ ] `deploy/**` contains only declarative deployment artifacts and static configuration; it contains no `.mjs`, `package.json`, lockfile, or `.npmrc`.
- [ ] The Cloud profile owns one nested manifest and lockfile at `cloud/`, with only Cloud-required dependencies and commands and no workspace/ancestor-hoist reach.
- [ ] The root manifest exposes only Host Edge, CI, and developer commands with current consumers; every retained script in either manifest is evidenced in the PR.
- [ ] Host profile source lives under `src/composition/orchestrator/` and is imported by the Host entrypoint; it is not copied into another tree.
- [ ] The OIDC and embedding mock servers live under test fixtures and are started only by tests.
- [ ] Docker, Compose, launchd, CI, tests, and current guides resolve from the new authorities.
- [ ] Focused package-boundary tests reject executable code or package authority under `deploy/**`.
- [ ] The Brain architecture map names the new composition role without preserving the legacy `ai/**` placement as authority.
- [ ] Existing Host CI remains green, including container-backed integration tests; changed specs are executed locally with `--workers=1` because the Brain Unit workflow does not execute the full collection.

## Out of Scope

- moving all Brain domains or the Orchestrator implementation;
- selecting the final task/adaptor closure for both profiles;
- creating a Cloud source tree or Cloud test runner;
- adding a dependency-injection container, registry, generated command catalog, or one-shot migration ledger;
- production deployment or release.

## Avoided Traps

- no source under `deploy/**`;
- no `cloud/src` or mirrored Host/Cloud domain tree;
- no root forwarding scripts into Cloud;
- no script retained solely because a historical migration or private test once used it;
- no second profile/config authority beside AiConfig.

## Related

Child of #213. Corrective successor to closed #197. #215 established the first canonical `src/**` domain slice; this ticket establishes the composition edge without reopening that domain work.

Origin Session ID: 01a03d1e-3f28-7350-adb6-0f176c546d63

Retrieval Hint: `query_raw_memories("Brain #214 deploy declarative cloud package hostEdgeProfile composition root")`


## Timeline

- 2026-08-28T22:16:17Z @neo-gpt-emmy added the `bug` label
- 2026-08-28T22:16:17Z @neo-gpt-emmy added the `ai` label
- 2026-08-28T22:16:17Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-28T22:16:17Z @neo-gpt-emmy added the `testing` label
- 2026-08-28T22:16:17Z @neo-gpt-emmy added the `architecture` label
- 2026-08-28T22:16:18Z @neo-gpt-emmy added the `build` label
- 2026-08-28T22:16:18Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-28T22:16:18Z @neo-gpt-emmy added the `tech-debt` label
- 2026-08-28T22:20:27Z @neo-gpt-emmy cross-referenced by #198
- 2026-08-28T22:22:06Z @neo-gpt-emmy cross-referenced by #192
- 2026-08-28T22:22:13Z @neo-gpt-emmy cross-referenced by #197
- 2026-08-28T22:32:03Z @neo-gpt-emmy cross-referenced by #216
- 2026-08-28T22:35:26Z @neo-opus-vega cross-referenced by #23
- 2026-08-29T17:48:28Z @neo-gpt-emmy cross-referenced by #215
- 2026-08-29T17:51:53Z @neo-gpt-emmy cross-referenced by PR #235
- 2026-08-29T18:52:12Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T19:32:07Z @neo-gpt-emmy cross-referenced by PR #236
- 2026-08-29T19:48:49Z @tobiu referenced in commit `0678d40` - "Merge pull request #236 from neomjs/codex/214-declarative-deployment

fix(architecture): make deployment declarative-only (#214)"
- 2026-08-29T19:48:49Z @tobiu closed this issue
- 2026-08-29T19:55:00Z @neo-opus-vega cross-referenced by #237
- 2026-08-29T21:49:51Z @neo-opus-vega cross-referenced by #239
- 2026-08-30T23:19:19Z @neo-opus-vega cross-referenced by #12
- 2026-09-21T10:38:48Z @neo-opus-grace cross-referenced by PR #397
- 2026-09-21T10:49:50Z @neo-opus-grace cross-referenced by #201

