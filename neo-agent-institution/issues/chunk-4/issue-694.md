---
id: 694
title: The unsupported-harness card test still picks Antigravity
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-opus-grace
createdAt: '2026-10-15T10:16:44Z'
updatedAt: '2026-10-15T12:30:03Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/694'
author: neo-opus-grace
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
closedAt: '2026-10-15T12:30:03Z'
---
# The unsupported-harness card test still picks Antigravity

## Context

Dependabot PRs #692 and #693 are red on one test, deterministically across both retries: `test/playwright/unit/apps/agentos/Accounts.spec.mjs:1160`, *"unsupported, disconnected, and missing saved targets stay visible but inert"*. It expects `Unavailable for this harness` for a seat with `harnessType: 'antigravity'`.

Since Brain `2b042f10` (neomjs/neo-agent-brain#1025, merged 2026-10-14T14:58Z, "boot Antigravity with seat MCP and memory"), the Fleet contract's `HARNESS_TYPES` (`src/fleet/contract/harnessTypes.mjs:27`) gives `antigravity` `tenantMcpTarget: true`. So `supportsTenantMcpTarget` (`:111`) no longer refuses it, and `AgentConfigComponent.getTenantChoiceAvailability` (`apps/agentos/view/fleet/detail/AgentConfigComponent.mjs:379–380`) renders no "unavailable" reason.

The CI runs show the timing exactly:
- #692 (Actions bumps only) passes its locked-dependency step, because the lock pins Brain `4ec5c03`, from before the change. It fails once the job resolves Brain at `dev`, in `Isolated Institution` and `Explicit Brain contract`.
- #693 bumps the lock to Brain `dfd9a3e`, so it fails even the locked step.
- `dev`'s last Institution CI (13:32Z at `afabdef`, green) predates the Brain merge.

So every new Institution PR now fails these two jobs. The dependabot pipeline itself is sound: every baseline and automerge job passes on both PRs.

## The Fix

Name a harness the contract still refuses as the test's unsupported harness. At Brain `dev` that is only `native-neo` (`tenantMcpTarget: false`). The test caught a real consumer-facing contract change, which is what it is for; it only names the wrong example now.

## Acceptance Criteria

- [ ] AC-1: The case passes against the locked Brain and against Brain at `dev` (`npm run resolve-org-dev`, then `npm run test-unit`).
- [ ] AC-2: Institution CI is green on the PR, `Isolated Institution` and `Explicit Brain contract` included.
- [ ] AC-3 (post-merge): rebased, #692 and #693 pass the same two jobs.

## Out of Scope

Dependabot workflow changes (none needed) and Brain's harness matrix (#1025 is the intended contract).

## Related

neomjs/neo-agent-brain#1025 · #692 · #693

Decision Record impact: none.

Live latest-open sweep: latest 20 open issues at 2026-10-15T10:15Z; no equivalent. A2A sweep: latest 15 messages; no claim. MC sweep: "Institution unit test unsupported harness Antigravity tenant MCP target…", no prior decision. Own-assignment sweep: no overlapping surface.

Origin Session ID: c8cf23fd-40af-4f29-963a-6fca621e3be8

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-15T10:16:44Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-15T10:16:46Z @neo-opus-grace added the `bug` label
- 2026-10-15T10:16:46Z @neo-opus-grace added the `ai` label
- 2026-10-15T10:18:00Z @neo-opus-grace cross-referenced by PR #695
- 2026-10-15T12:30:03Z @tobiu referenced in commit `44e1410` - "test(agentos): the unsupported-harness case names native-neo (#694) (#695)"
- 2026-10-15T12:30:03Z @tobiu closed this issue

