---
id: 1370
title: The Brain unit suite inherits a seat's plane placement and git identity
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-ada
createdAt: '2026-10-30T00:55:01Z'
updatedAt: '2026-10-30T01:37:01Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1370'
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
closedAt: '2026-10-30T01:37:01Z'
---
# The Brain unit suite inherits a seat's plane placement and git identity

## Context

Measured 2026-10-30 in an FM-launched seat's Brain clone at `dev@98603a40`. Ten unit tests fail there that pass in CI; Brain Unit is green on `dev`. An isolated run of their eight files on base `b6f24069` fails the same ten.

I re-ran the eight files with every inherited `NEO_*`, `MCP_*`, `GIT_AUTHOR_*` and `GIT_COMMITTER_*` variable unset, and verified that none were left. 9 of the 10 then pass. Eight of those are deterministic, listed below. The ninth, a `MemoryCoreRecorderService` arm, varies by run with or without the environment. Only the `McpServersHealth` server boots still fail, now for all three servers.

Defect-note `a559049a`. Its first falsification line was a zsh no-op, corrected in `6affcb95`; this body is the record.

## The Problem

A seat's process environment carries the Fleet Manager's runtime placement: about 30 `NEO_*` names, among them `NEO_PLANE_DATA_ROOT`, `NEO_CHROMA_DATA_DIR`, `NEO_AI_DB_PATH` and the `NEO_HEARTBEAT_*` paths. It also carries the seat's git identity in `GIT_AUTHOR_*` and `GIT_COMMITTER_*`. The unit suite inherits all of it:

- **`config.template.spec` "ships Tier-1 provider and unified Chroma defaults" and `memory-core/config.template.spec` "inherits deployment-wide Tier-1 defaults".** `engines.chroma.dataDirProd` resolves to FM's live plane (`~/Library/Application Support/neo-harness/brain/chroma/unified`) instead of the checkout default.
- **`planeConfig.spec` "a Provider with ONLY NEO_PLANE_DATA_ROOT changed fails member coherence" and `BaseServer.spec` "composed Tier-1 claims fail a relocated boot".** The inherited member placements count as explicitly placed, so coherence does not throw.
- **`checkCommitAuthorship.spec`, four arms.** The inherited identity overrides the one each arm plants in its fixture repository, so the operator commit is never seen.

Every local verification in a seat therefore starts from ten reds, which need an isolated base run to rule out. CI has none of these variables, so it never sees the difference.

## The Architectural Reality

- **The harness binds its own values but removes none.** `test/playwright/playwright.config.unit.mjs` sets `UNIT_TEST_MODE` and a few `NEO_*` values. `test/playwright/configTemplateResolver.mjs` binds the test storage paths (`NEO_MEMORY_LOG_PATH` through `NEO_RECOVERY_ACTUATOR_HEAL_ATTEMPTS_PATH`) before the first `ConfigProvider` import. That is ADR 0019 §5.4 rung (ii), the sanctioned place for worker-wide test environment. Nothing removes inherited placement that the harness does not bind.
- **Only the prod leaves move.** Under `UNIT_TEST_MODE` the effective paths stay on test roots, as the failing spec's own diff shows: `dataDir` is the tmp root. Whether any test reads a prod leaf to act on it is unverified.
- **A scrub must respect the boundary.** Specs spawn children that inherit the worker environment, including the `NEO_TEST_*` values the resolver reads as an inherited boundary (`NEO_TEST_CONFIG_TEMPLATES`, `NEO_TEST_STORAGE_ROOT`). So a scrub runs once, at the top of the tree, before the harness binds its own values, and never inside the boundary.
- **Some specs need the developer's live substrate.** CI skips them through `NEO_TEST_SKIP_CI`, and in a seat they reach that substrate through endpoint and credential variables. `McpServersHealth`'s remote boots need `NEO_MCP_REMOTE_TOKEN`, which the tracked client config requires, and three `SessionService` arms reach the model host. Removing every `NEO_*` name, which I measured, turned those five local passes red.

## The Fix

- **In `playwright.config.unit.mjs`, before the resolver binds anything:** remove inherited placement and git identity, unless the process already sits inside the harness boundary. Placement means the plane identity (`NEO_PLANE_ID`) and every path-valued `NEO_*` name (`*_PATH`, `*_DIR`, `*_ROOT`, `*_BASE`). Git identity means `GIT_AUTHOR_*` and `GIT_COMMITTER_*`.
- **`NEO_TEST_*`, endpoints, credentials and the agent identity stay.** The harness's own bindings follow, unchanged.
- **The scrub states its reason beside it:** a unit run behaves as CI does wherever it starts.

## Acceptance Criteria

- [ ] AC-1: in a seat checkout with the FM environment set, the eight deterministic arms above pass without unsetting anything by hand.
- [ ] AC-2: a spec pins the scrub's rule, its no-op inside the harness boundary, and its import before the resolver. Under the rule, placement and git identity go, while `NEO_TEST_*`, endpoints, credentials and the agent identity stay.
- [ ] AC-3: Brain Unit's head-vs-base comparison introduces no failure.

## Out of Scope

- `McpServersHealth`'s local neural-link boot failure, which fails the same way with and without the seat environment.
- The `MemoryCoreRecorderService` arm that varies by run.
- CI, which carries none of these variables.

## Related

#201 (the retained suite in CI), #571 (the seat layout), defect-note `a559049a`. Decision Record impact: aligned-with ADR 0019 §5.4, rung (ii).

Ownership: self-assigned. Reproducing AC-1 needs an FM-launched seat.

Live latest-open + closed sweep: latest 20 open issues, plus all-state searches for "unit suite inherited environment seat", "NEO_PLANE_DATA_ROOT unit test" and "GIT_AUTHOR checkCommitAuthorship", at 2026-10-30T00:54Z, with no equivalent. #273 (unit Chroma reaping) is the nearest closed hit, on another surface. A2A: no claim. Memory Core: no prior report of this cause.

Origin Session ID: be08763b-ad29-4a7b-8f90-a8e06b01b8de

Retrieval Hint: "Brain unit suite seat environment NEO_PLANE_DATA_ROOT GIT_AUTHOR inherited local reds CI green scrub"


## Timeline

- 2026-10-30T00:55:01Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-30T00:55:02Z @neo-opus-ada added the `bug` label
- 2026-10-30T00:55:03Z @neo-opus-ada added the `ai` label
- 2026-10-30T00:55:03Z @neo-opus-ada added the `testing` label
- 2026-10-30T01:10:17Z @neo-opus-ada cross-referenced by PR #1371
- 2026-10-30T01:37:01Z @tobiu referenced in commit `a4e2416` - "test(playwright): a unit run sheds the placement and git identity it inherits (#1370) (#1371)

isolateUnitEnvironment.mjs runs before configTemplateResolver.mjs binds the harness values. It removes the inherited plane identity, path-valued NEO_* names (*_PATH, *_DIR, *_ROOT, *_BASE) and GIT_AUTHOR_*/GIT_COMMITTER_*, so a seat's Fleet Manager placement no longer moves the config's prod leaves or counts plane members as placed. It also stops the seat's identity overriding a fixture's. NEO_TEST_*, endpoints, credentials and the agent identity stay: the specs CI skips for want of live substrate reach the developer's servers through them. Workers inherit the harness boundary and skip the scrub."
- 2026-10-30T01:37:02Z @tobiu closed this issue

