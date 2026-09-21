---
id: 347
title: Perspective tools mirror the engine's declared-perspective contract
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-09-12T14:07:53Z'
updatedAt: '2026-09-13T10:23:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/347'
author: neo-fable
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
closedAt: '2026-09-13T10:23:48Z'
---
# Perspective tools mirror the engine's declared-perspective contract

Brain half of the Neural Link perspective-tool leaf of neomjs/neo#18605 (the dock Workspace's reactive selection, graduated from neo Discussion #18594); the engine half is neomjs/neo#18613. Sequenced after the engine half: this ticket mirrors whichever contract that PR pins. `Decision Record impact: none`.

`unowned-rationale:` filed from the design seat at graduation; claimable by self-selection, ideally by whoever takes neomjs/neo#18613.

## Context

The engine's `list_perspectives` / `restore_perspective` client (`src/ai/client/DockService.mjs` in the engine, `:363` / `:413` at `dev@28e56e1543`) resolves stored records only. Under neomjs/neo#18605 a dock Workspace may declare its perspectives and select one with a reactive `activePerspective_`, and it publishes `dock.perspective.active`, `modified` and `pending` as provider data. An agent driving such a workspace through the Neural Link today sees an empty perspective list and cannot select a declared one.

## The Problem

The Brain's wire contract for the two tools is stored-record shaped: `ai/mcp/server/neural-link/openapi.yaml` (`operationId: list_perspectives` at `:944`, `restore_perspective` at `:974` at Brain `95bea64`), the wiring in `ai/mcp/server/neural-link/toolService.mjs`, the service pass-through in `ai/services/neural-link/DockService.mjs`, and the specs `test/playwright/unit/ai/services/neural-link/DockService.spec.mjs` and `test/playwright/unit/ai/mcp/validation/OpenApiValidatorCompliance.spec.mjs`. The capability matrix cited in the source Discussion is located: it is an **engine** document, `learn/agentos/tooling/NeuralLinkCapabilityMatrix.md` in neomjs/neo (rows for `list_perspectives` and `restore_perspective` under "Verb Matrix"), which this repository's `CapabilityMatrix.spec.mjs` reads by a path relative to its own root. Its two rows are updated under the engine's guides leaf neomjs/neo#18614 (author fold), not here.

## The Fix

Mirror the engine leaf's **shipped** contract (neomjs/neo#18634, engine dev@02cb33eae5 — amended at claim from the pre-intake row shape):

- **Enumeration:** `list_perspectives` gains `declared: String[]` (the names the workspace's `activePerspective` accepts) and `perspective: {active, modified, pending}` beside the unchanged stored summaries and keyed topologies — the key is the discriminator, since a declared name has no record to summarize; the fail-closed refusal fires only when the holder declares nothing and holds no store. `restore_perspective` resolves a name across the declared list and both record collections, refuses a name found in more than one with the sources named, and takes the workspace's accepted `activePerspective` write for a declared name (the same identity and provenance as a UI switch; the active name re-applies its baseline), reporting `source: 'declared'` and `schema: null`.
- The wire stays additive: both responses remain `type: object` without properties (the compliance spec's output-schema rule), so the contract lives in the descriptions and the service docblocks; the `toolService` wiring is unchanged (same two operations, same request shapes).

## Acceptance Criteria

- [ ] The OpenAPI descriptions of both tools and the service pass-through docblocks state the shipped contract (the declared list under `declared`, the facts under `perspective`, the declared restore path with `source`, the tie refusal); the request shapes and the wiring are unchanged; the compliance spec is green.
- [ ] `DockService.spec.mjs` keeps its dispatch arms and pins the descriptions' contract words, so a drift between the wire description and the engine contract fails a Brain test.
- [ ] The tool descriptions name the published facts an agent can read after a restore.
- [ ] The PR `Refs neomjs/neo#18605` and `Resolves` this ticket only; it opens after neomjs/neo#18613 merges.

## Out of Scope

The engine client (neomjs/neo#18613); saved-name selection through the config (deferred on the epic).

## Related

neomjs/neo#18605 · neomjs/neo#18613 · neo Discussion #18594 (STEP_BACK row 2).

Live latest-open sweep (this repository): checked the latest 20 open issues at 2026-09-12T14:07:15Z; no equivalent found.

Origin Session ID: c42870ab-3721-40f0-a1f8-f385786ffc3c
Retrieval Hint: "neural-link list_perspectives restore_perspective declared perspectives source discriminator"

🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session c42870ab-3721-40f0-a1f8-f385786ffc3c


## Timeline

- 2026-09-12T14:07:55Z @neo-fable added the `enhancement` label
- 2026-09-12T14:07:55Z @neo-fable added the `ai` label
- 2026-09-12T14:07:55Z @neo-fable added the `agent-os` label
- 2026-09-12T14:08:30Z @neo-fable cross-referenced by #18613
- 2026-09-12T16:55:11Z @neo-gpt cross-referenced by #348
- 2026-09-12T21:49:15Z @neo-fable cross-referenced by PR #18634
- 2026-09-12T22:18:16Z @neo-fable assigned to @neo-fable
- 2026-09-12T22:23:34Z @neo-fable referenced in commit `e4d1d42` - "docs(neural-link): perspective tools state the declared contract (#347)

The engine's list_perspectives and restore_perspective now see declared
perspectives (neomjs/neo#18613, merged as neomjs/neo#18634). The Brain's
wire contract is a pass-through, so the mirror is the description: the
OpenAPI rows and the service docblocks state the shipped shape — the
declared list under `declared` with the published perspective facts
under `perspective` beside the untouched stored summaries, the declared
restore on the workspace's accepted activePerspective write reporting
source 'declared', and the tie refusal naming its sources. Request
shapes, wiring and the bare object responses are unchanged, so the
compliance rule for output schemas holds.

Spec: the dispatch arms stay; one arm parses the OpenAPI document and
pins the contract words in both descriptions, so a drift between the
wire description and the engine fails a Brain test."
- 2026-09-12T22:23:36Z @neo-fable cross-referenced by PR #350
- 2026-09-13T09:30:45Z @neo-fable referenced in commit `b1448cf` - "docs(neural-link): the list refusal condition is non-exclusive (#347)

list_perspectives said it fails closed ONLY when the holder declares nothing and
exposes no store. The engine also returns a topology collection's validation errors
while a perspective store exists (DockService.listPerspectives, topologyErrors branch),
so the absence condition is one refusal among two: the description, the service
docblock and the description-pinning arm now name both."
- 2026-09-13T10:23:48Z @tobiu referenced in commit `f50e93e` - "Merge pull request #350 from neomjs/feature/347-perspective-tools-declared-contract

docs(neural-link): perspective tools state the declared contract (#347)"
- 2026-09-13T10:23:48Z @tobiu closed this issue

