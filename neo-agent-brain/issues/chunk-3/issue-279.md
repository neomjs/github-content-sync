---
id: 279
title: DeploymentCookbook still tells operators to disable `kbSync` in cloud — a lane the config now defaults ENABLED and whose env pin a spec refuses
state: CLOSED
labels:
  - bug
  - documentation
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-08-31T07:25:16Z'
updatedAt: '2026-08-31T07:50:28Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/279'
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
closedAt: '2026-08-31T07:50:28Z'
---
# DeploymentCookbook still tells operators to disable `kbSync` in cloud — a lane the config now defaults ENABLED and whose env pin a spec refuses

## The problem

`learn/agentos/DeploymentCookbook.md` §2 is the operator-facing rendering of ADR-0014's lane taxonomy. ADR-0014 and the runtime authority map have both moved `kbSync` to the container plane; the Cookbook has not. It now instructs an operator to do the opposite of what the code does, in three places.

The Cookbook opens §2 with *"ADR 0014 classifies every orchestrator scheduler lane…"*, so it presents itself as a faithful projection of that ADR. It is currently a stale copy of it.

## Evidence (all read at `origin/dev`)

**Authority — `ai/daemons/orchestrator/taskAuthority.mjs`:**

```
:70   **`kbSync` and `temporal-summary` are container-plane, not host-edge**
:101  kbSync             : ORCHESTRATOR_AUTHORITY_CLASS.containerPlane,
:105  'temporal-summary' : ORCHESTRATOR_AUTHORITY_CLASS.containerPlane,
:92   bridgeDaemon       : ORCHESTRATOR_AUTHORITY_CLASS.hostEdge,
:96   mlx                : ORCHESTRATOR_AUTHORITY_CLASS.hostEdge,
:107  'primary-dev-sync' : ORCHESTRATOR_AUTHORITY_CLASS.hostEdge,
```

**ADR-0014 `:203`** already records the reclassification: *"That is **no longer** its authority class — `TASK_AUTHORITY_BY_NAME` has both `kbSync` and `temporal-summary` as `container-plane`, because the container *is* the checkout."*

**Config — `ai/configBase.mjs`:** `kbSyncEnabled` sits in the **`cloudOnly`** group, documented as *"`null` means use the deployment-profile default — **cloud enables**, local disables."* In a cloud deployment `kbSync` therefore defaults **ENABLED**.

**The three false claims in `DeploymentCookbook.md`:**

| # | Location | Claim | Reality |
|---|---|---|---|
| 1 | §2 lane table (`:44`) | `` `bridgeDaemon`, `mlx`, `kbSync`, `primary-dev-sync` `` → *"Local-only lanes. They must be disabled in a tenant cloud deployment."* | Only `kbSync` is misplaced; the other three are correctly host-edge. `kbSync` is container-plane. |
| 2 | §2 env-override table | `NEO_ORCHESTRATOR_KB_SYNC_ENABLED=false` listed under *"Cloud default intent"* | The cloud default is **enabled**. Worse: `configBase.mjs:2069-2072` states `mcpHealthcheck.spec.mjs` **refuses a compose that pins `NEO_ORCHESTRATOR_KB_SYNC_ENABLED`**, "because a deployment restating an AiConfig default silently freezes today's value." The Cookbook recommends a pin the test substrate rejects. |
| 3 | §2 Sub D paragraph | *"cloud-mode default resolution disables `primary-dev-sync`, `kbSync`, `bridgeDaemon`, and Golden Path repo enrichment"* | True for all except `kbSync`. |

## Why this matters

`kbSync` owns the image-carried **shared Neo corpus** scan. `configBase.mjs` states plainly why the leaf lives in `cloudOnly` rather than `localOnly`: *"a `localOnly` leaf resolves to disabled on the only role left that can [run it], leaving the Knowledge Base with no producer at all."* An operator following the Cookbook disables the only producer of the shared corpus in exactly the deployment the guide is written for.

This is a correctness defect in live guidance, not a wording preference.

## Scope boundary

**This is deliberately NOT the deployment-guide rewrite.** #86 covers rewriting the guide tree for onboarding and is explicitly blocked on runtime parity (#90) and config-default consolidation. This ticket corrects three factually-wrong statements that should not stay live while #86 waits. Narrow by construction: no restructuring, no length reduction, no section moves.

`bridgeDaemon`, `mlx`, and `primary-dev-sync` are **correct as written** and must not be touched — the fix is `kbSync`-only. `temporal-summary` is absent from the Cookbook table entirely; adding it is optional and explicitly out of scope for the blocking ACs.

## Acceptance Criteria

- **AC-1** — §2's lane table no longer classifies `kbSync` as a local-only lane that must be disabled in a tenant cloud deployment, and instead reflects its container-plane authority class. `bridgeDaemon`, `mlx`, `primary-dev-sync` remain classified exactly as they are today.
- **AC-2** — The `NEO_ORCHESTRATOR_KB_SYNC_ENABLED=false` row no longer presents disabling `kbSync` as cloud default intent. If the variable is still documented, the entry states that the cloud default is enabled and that pinning it in compose is refused by `mcpHealthcheck.spec.mjs`.
- **AC-3** — The Sub D paragraph no longer asserts that cloud-mode default resolution disables `kbSync`.
- **AC-4** — A grep for the retired framing finds no surviving instance in `DeploymentCookbook.md`: no line associates `kbSync` with "local-only" or with a cloud-deployment disable instruction. The check is run with a positive control so a clean result is not a blind one.
- **AC-5** — No source, config, or spec file is modified. This is a documentation correction only; if the doc and the code disagree anywhere else, that becomes a separate ticket rather than a code change here.

## Provenance

Surfaced while reviewing PR #276 (#263). I initially raised it as a required action against that PR and **retracted it** — the drift predates #276, which neither introduced nor widened it. Recording the correction here because the finding was true while its attribution was not: I inferred the cause without first checking whether `dev` already carried the amendment.

Related: #86 (blocked guide rewrite), #263, PR #276, ADR-0014, ADR-0019.

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5 · Claude Code

## Timeline

- 2026-08-31T07:25:16Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-08-31T07:25:17Z @neo-opus-ada added the `bug` label
- 2026-08-31T07:25:18Z @neo-opus-ada added the `documentation` label
- 2026-08-31T07:25:18Z @neo-opus-ada added the `ai` label
- 2026-08-31T07:25:18Z @neo-opus-ada added the `agent-os` label
- 2026-08-31T07:28:24Z @neo-opus-ada cross-referenced by PR #280
- 2026-08-31T07:50:28Z @tobiu closed this issue

