---
id: 324
title: FleetServerComposition fixtures lack the orchestrator section (3 arms red)
state: CLOSED
labels:
  - bug
  - ai
assignees:
  - neo-fable-clio
createdAt: '2026-09-05T00:54:43Z'
updatedAt: '2026-09-05T11:58:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/324'
author: neo-fable-clio
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
closedAt: '2026-09-05T11:58:40Z'
---
# FleetServerComposition fixtures lack the orchestrator section (3 arms red)

## Context

Running the fleet unit tree on `dev@442a220` for #323 shows three `FleetServerComposition.spec.mjs` arms red (`:265`, `:452`, `:595`), all with the same throw:

```
TypeError: Cannot read properties of undefined (reading 'deploymentStateBridge')
    at ai/services/fleet/fleetServer.mjs:1273
```

Observed on the tree run (`npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/services/fleet`: 710 passed, 3 failed, 4 did not run) and again on the spec alone (19 passed, 3 failed). The tree is red on `dev` today.

## The Problem

`startFleetServer` (#314, PR #315) wires the deployment-state read seam from the resolved config — `aiConfig.orchestrator.deploymentStateBridge.{snapshotPath, staleAfterMs, maxSnapshotBytes}` (`fleetServer.mjs:1271-1276`). The three arms hand-build their `aiConfig` literal with `publicUrl`, `mcpHttpHost`, `mcpListenHost` and `fleet` only (`FleetServerComposition.spec.mjs:310`, `:484`, `:616`); the literals predate the seam (last touched by #13 / #197 / #225). The production read is correct: ADR-0019 B3 forbids a defensive `?.` on a config read — the SSOT guarantees the tree, so the fixture must carry the section.

Observation: the throw and the three red arms. Inference: PR #315's own arms (`fleetServer.spec.mjs:837`, `:1006`) supply the section and pass, which is why the wiring shipped green while the composition arms were not in that PR's run scope.

## The Architectural Reality

- `learn/agentos/decisions/0019-aiconfig-reactive-provider-ssot.md` §3 B3 (no defensive `?.`; fail loud) and C3 (tests import the canonical template — these fixtures are literals, a broader cleanup than this ticket).
- The seam's own contract (`fleetServer.mjs:1271`): "an empty path leaves the seam unwired and the log says which" — fail-soft by construction, so a fixture can carry the section without a snapshot file.

## The Fix

Add `orchestrator: {deploymentStateBridge: {snapshotPath: '', staleAfterMs: 120_000, maxSnapshotBytes: 256 * 1024}}` to the three `startFleetServer` configs in `FleetServerComposition.spec.mjs`. No production change; no `?.` added to the read.

## Acceptance Criteria

- [ ] `FleetServerComposition.spec.mjs` arms `:265`, `:452`, `:595` green on the branch in the same command that shows them red on `dev@442a220`; the rest of the `test/playwright/unit/ai/services/fleet` tree is unchanged by the fix — measured at the PR head as 714 passed · 5 failed · 4 did not run, where the nine non-green rows are checkout environment identical on `dev` (`provisioningTemplates.spec.mjs` `:72` `:79` `:90` `:115` on the untracked `.codex` / `.claude` templates; `fleetServer.spec.mjs:818` on a `remRunStateDir` member this checkout's resolved config predates, with its four serial siblings skipped behind it). No hosted job executes the fleet tree (`brain-unit.yml` runs four smoke specs), so the receipt is the exact-head local command on the PR. *(Restated 2026-09-05 from "the whole tree green on the branch" — Emmy's evidence RA on PR #325.)*
- [ ] `fleetServer.mjs:1273` unchanged (no defensive read).

## Out of Scope

Converting the composition spec's literals to the canonical template (ADR-0019 C3, a separate cleanup) · the projection shape (#323, same PR).

Decision Record impact: aligned-with ADR 0019 (B3).

## Related

#314 (the seam) · PR #315 (shipped the read) · #323 (sibling follow-up, delivered in the same PR).

Live latest-open sweep: latest 20 open Brain issues read 2026-09-05T00:47:37Z, no equivalent. A2A in-flight claim sweep: mailbox listing 2026-09-05T00:36Z, no claim on this surface. Memory Core sweep (`query_raw_memories`, the TypeError symptom) returned no prior decision. Own-assignment sweep: my open Brain tickets (#323, #322, #318, #37, #50, #51, #53) — #323 is the sibling.

Origin Session ID: 49133900-1f86-4134-a82b-30ff0709bcaf

Retrieval Hint: `query_raw_memories("FleetServerComposition orchestrator deploymentStateBridge fixture undefined")`

📜 Clio · @neo-fable-clio · Claude Fable 5.1 · Claude Code · session 49133900-1f86-4134-a82b-30ff0709bcaf


## Timeline

- 2026-09-05T00:54:43Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-05T00:54:44Z @neo-fable-clio added the `bug` label
- 2026-09-05T00:54:44Z @neo-fable-clio added the `ai` label
- 2026-09-05T00:57:49Z @neo-fable-clio cross-referenced by PR #325
- 2026-09-05T01:06:07Z @neo-fable-clio cross-referenced by #323
- 2026-09-05T11:35:06Z @neo-gpt-emmy cross-referenced by PR #114
- 2026-09-05T11:58:40Z @tobiu closed this issue

