---
id: 1194
title: 'A fresh seat''s Start fails on a packaged FM: configs copy read-only'
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-10-19T14:06:46Z'
updatedAt: '2026-10-19T16:25:53Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1194'
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
closedAt: '2026-10-19T16:25:53Z'
---
# A fresh seat's Start fails on a packaged FM: configs copy read-only

## Context

neomjs/neo-agent-institution#736's pre-install rehearsal ran its first packaged candidate on 2026-10-19 (Brain `cf0c1a1e`, Institution `ec61be4`). Both fresh stand-in seats were refused at Start with `FLEET_WORKSPACE_PREPARATION_FAILED`. The Fleet log names the cause:

```
prepareManagedAgentWorkspace: host apply effect failed. · cause: EACCES: permission denied, open '<smoke>/fleet/agents/rehearsal-claude-desktop/rehearsal/seat/ai/mcp/server/github-workflow/config.mjs'
```

The file had been created moments earlier, mode `-r--r--r--`. The same journey passes in checkout mode (#1165's unit job, the checkout rehearsal).

## The Problem

`bootstrapWorktree` (`ai/scripts/migrations/bootstrapWorktree.mjs`) fills a seat checkout's missing configs from the runtime root: `fs.copyFile(src, dst)`, then for a per-server config `fs.writeFile(dst, materializeServerConfigTemplate(...))`. `copyFile` carries the source's mode. A packaged FM's runtime root is its materialized runtime generation, whose files are read-only, so the copy is read-only and the write on the next line fails. A checkout's runtime root is writable, which is why CI and checkout runs never see it.

Measured: a fresh seat on a packaged candidate. Inference, not verified on the installed app: a new seat there takes the same path. Existing seats skip the copy, since their files already exist.

Design authority: Institution [ADR 0034 §2.5, Runtime generations for managed seats](https://github.com/neomjs/neo-agent-institution/blob/dev/learn/agentos/decisions/0034-electron-shell-architecture.md#runtime-generations-for-managed-seats) items 1–2 and neomjs/neo-agent-institution#674's first Contract Ledger row make a runtime generation an immutable, verified source, whose evidence covers executable modes. [ADR 0040 §2.5](https://github.com/neomjs/neo-agent-brain/blob/dev/learn/agentos/decisions/0040-agentos-extraction-topology.md) (`agentosRuntimeRoot` vs `targetRepoRoot`) and `bootstrapWorktree`'s JSDoc ("into the target project root", "the copied operator overlay") place the copied configs in the seat's own checkout. The copy carried the source's mode across that boundary. This ticket corrects the copy, not either record.

## The Architectural Reality

- `prepareManagedAgentWorkspace` hydrates through `hydrateCurrentWorktree` → `bootstrapWorktree({mainCheckout: <runtime root>, projectRoot: <seat checkout>})`.
- A packaged runtime generation is created read-only by design (`dr-x` directories, `r--` files under `userData/brain/runtime/<generation>/`), and a seat must never be able to edit it.
- The copied file belongs to the seat afterwards: it is the seat's config, materialized and later rewritten by its own setup.

## The Fix

Copy the bytes, not the mode: write each bootstrapped config from the source's contents (`fs.readFile` → `fs.writeFile`), so it is created with the default mode like any file the seat writes. The per-server materialization then writes over a file the seat owns.

## Acceptance Criteria

- [ ] AC-1 (red first): `bootstrapWorktree.spec.mjs` copies from a read-only source tree; a per-server config is materialized and every copied file is writable by its owner. Red on `dev` (EACCES), green at head.
- [ ] AC-2: Existing arms keep passing (skip-when-present, missing source, materialization).

The packaged observation, both fresh seats starting on a candidate with this fix, is neomjs/neo-agent-institution#736's own AC-2. This ticket does not carry it.

## Out of Scope

- The runtime generation's own permissions: read-only is correct there.
- Seats whose checkout already carries the files: unchanged.

## Related

neomjs/neo-agent-institution#736 (found it) · neomjs/neo-agent-institution#732 (FM auto-update) · #1164 (the journey the rehearsal runs)

Live latest-open sweep: latest 20 open Brain issues read 2026-10-19T14:06Z, no equivalent. Org search (`bootstrapWorktree`, `EACCES config.mjs`, `FLEET_WORKSPACE_PREPARATION_FAILED`): nothing equivalent. MC sweep: `bootstrapWorktree copyFile read-only runtime generation per-server config.mjs EACCES fresh seat`, 6 results, no prior decision. Own-assignment sweep: #571 is adjacent (seat folder layout), not overlapping.

Origin Session ID: 17ba709b-6759-4aa3-b484-e78a178b7f19
Retrieval Hint: "packaged FM fresh seat EACCES bootstrapWorktree copyFile mode"

## Timeline

- 2026-10-19T14:06:46Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-19T14:06:48Z @neo-opus-ada added the `bug` label
- 2026-10-19T14:06:48Z @neo-opus-ada added the `ai` label
- 2026-10-19T14:06:49Z @neo-opus-ada added the `agent-os` label
- 2026-10-19T14:10:40Z @neo-opus-ada referenced in commit `959c554` - "docs(fleet): bootstrapWorktree's comments cite ADRs with the decision-record escape and keep ticket history out (#1194)"
- 2026-10-19T14:10:48Z @neo-opus-ada cross-referenced by PR #1195
- 2026-10-19T14:17:11Z @neo-opus-ada referenced in commit `3361d64` - "chore(fleet): bring dev into the #1194 branch, so the packaged rehearsal runs the journey with this fix (#1194)"
- 2026-10-19T16:25:53Z @tobiu closed this issue
- 2026-10-19T16:25:53Z @tobiu referenced in commit `a4b18f7` - "fix(fleet): a fresh seat's configs copy the runtime root's bytes, never its read-only mode (#1194) (#1195)

* fix(fleet): a fresh seat's configs copy the runtime root's bytes, never its read-only mode (#1194)

bootstrapWorktree copied each missing config with fs.copyFile, which carries
the source's mode. A packaged FM's runtime generation is read-only, so the
copy was too, and the per-server materialization on the next line failed
EACCES: Start refused FLEET_WORKSPACE_PREPARATION_FAILED for a fresh seat.
Found by Institution #736's first packaged rehearsal.

* docs(fleet): bootstrapWorktree's comments cite ADRs with the decision-record escape and keep ticket history out (#1194)"

