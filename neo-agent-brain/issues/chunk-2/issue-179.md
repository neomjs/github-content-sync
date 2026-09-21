---
id: 179
title: Re-pin Brain and close residual Engine AI imports
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-26T21:14:09Z'
updatedAt: '2026-08-26T23:08:11Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/179'
author: neo-gpt-emmy
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
closedAt: '2026-08-26T23:08:11Z'
---
# Re-pin Brain and close residual Engine AI imports

## Context

[Brain PR #178](https://github.com/neomjs/neo-agent-brain/pull/178) merged at `68870d8c994bb6c1bc13d65f75cdd0929ff51517`, receiving the Agent OS against Neo `dev@e7874db2d2885a04314a109bdb5132753ae3cb49`. Before the merge completed, Neo advanced to `21da68021a4ccfa3d8d368972e646cb8a5644a30` through the skills-package cut.

Live latest-open sweep: checked the newest 20 Brain issues plus the latest 30 A2A messages at 2026-08-26T21:13:29Z; no equivalent follow-up exists. Semantic recall also returned no matching closure ticket.

## The Problem

The merged Brain is not yet deletion-ready:

- its manifest and lock still pin Neo `e7874db2…`, not current `dev@21da6802…`;
- the current Neo receive set is two paths smaller because the skill-manifest lint and its spec moved to `neo-agent-skills`;
- `ProtoSource.mjs` imports two Brain modules through `neo.mjs/ai/**`;
- the copied external pre-push hook resolves two Brain CLIs through `node_modules/neo.mjs/ai/**`;
- Brain `dev` has the unit and OpenAPI jobs, but six planned Brain-owned enforcement workflows plus integration/parity execution are still absent.

Those four consumers work only because the pre-deletion Engine package still carries `ai/**`. Once neomjs/neo#17791 deletes that tree and Brain refreshes its pin, they fail.

## The Architectural Reality

Brain-local runtime modules resolve from this repository's `ai/**` tree. The temporary `neo.mjs` dependency owns Engine/Body surfaces only; it must never be a fallback distribution channel for Brain code.

An external workspace hook cannot use Brain-relative paths. It must receive one explicit, absolute Agent OS runtime root and fail before a push when that authority or either CLI is missing.

The scoped Agent OS structure map completed successfully. Owning siblings remain:

- `ai/services/knowledge-base/source/Base.mjs`;
- `ai/mcp/server/knowledge-base/config.mjs` (generated from the local template lifecycle);
- `ai/scripts/maintenance/{kbPushClient,ingestTenant}.mjs`;
- `ai/examples/cloud-deployment/pre-push-hook.sh`.

## The Fix

1. Refresh the exact received populations from Neo `dev@21da6802…`, including deletion of:
   - `ai/scripts/lint/lint-skill-manifest.mjs`;
   - `test/playwright/unit/ai/scripts/lint/lintSkillManifest.spec.mjs`.
2. Re-pin `package.json` and `package-lock.json` to the same immutable Neo SHA.
3. Repoint `ProtoSource.mjs` to its local Brain modules by relative path.
4. Make `pre-push-hook.sh` require an absolute `NEO_AGENTOS_RUNTIME_ROOT`, derive both CLIs below it, and fail loud when the root or files are invalid.
5. Update the worked-example README commands to the same runtime-root contract.
6. Prove no `neo.mjs/ai/**` or `node_modules/neo.mjs/ai/**` residue remains before releasing neomjs/neo#17791.
7. Receive the six Brain-owned enforcement workflows named by neomjs/neo#17783 and add one Brain integration/parity job for the already-received suites.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Engine dependency pin | current Neo `dev` SHA | manifest and lock resolve `21da6802…` exactly | mutable branch or stale SHA fails | PR body | package/lock read-back |
| Brain example imports | Brain repository tree | import local Source base and KB config | no Engine-package fallback | example source | zero-residue scan + import collection |
| External pre-push hook | `NEO_AGENTOS_RUNTIME_ROOT` | require absolute root; derive both CLIs | absent, relative, or missing CLI exits non-zero | example README | shell positive/negative controls |
| Received source set | Neo `dev@21da6802…` | 1,693 exact identities before the explicit closure deltas | missing, extra, or substituted path fails | PR body | set and blob comparison |
| Brain receive-side CI | neomjs/neo#17783 subject-custody plan | receive the six named enforcement wrappers and run the received integration/parity suites | no Engine-side wrapper fallback | workflow files | workflow inventory + CI runs |

## Decision Record impact

Aligned with ADR 0040's receive-before-remove and Engine-independence boundary. This closes a post-merge timing delta; it does not amend topology.

## Acceptance Criteria

- [ ] The current receive populations match Neo `21da6802…`: 839 `ai/**`, 789 unit-AI, 54 integration, 3 integration-parity, 1 test-`ai`, 1 restore adapter, and 6 Brain benefit guides; no missing or extra identities.
- [ ] Only the existing Brain install binding plus the explicit consumer-closure edits differ from source bytes.
- [ ] `package.json`, the lock root, and the resolved `neo.mjs` package identify the same full `21da6802…` SHA.
- [ ] Repository-wide executable/example scan finds zero `neo.mjs/ai/**` and zero `node_modules/neo.mjs/ai/**` references.
- [ ] `ProtoSource.mjs` collects against local Brain modules.
- [ ] The hook accepts an absolute valid `NEO_AGENTOS_RUNTIME_ROOT` and rejects absent, relative, or incomplete roots before invoking Node.
- [ ] Fresh install, prepare/config materialization, complete unit collection, focused smoke, and required CI are green.
- [ ] Brain contains the six planned enforcement workflows plus a Brain integration/parity workflow; their referenced scripts/configs resolve locally.
- [ ] neomjs/neo#17791 remains blocked until this Brain PR merges.

## Out of Scope

- Engine-side deletion;
- Host/Cloud topology changes;
- Fleet Manager/Institution extraction;
- republishing Neo or `neo-agent-skills`;
- reopening closed Brain issue #13.

## Avoided Traps

- **Stale package masks local dependency:** zero-residue proof runs before the Engine pin advances.
- **Cwd fallback for external hooks:** one explicit absolute runtime authority only.
- **Branch-only pin:** manifest and lock bind the immutable full SHA.
- **Reopening resolved work:** this is a new post-merge timing successor.

## Related

- [Merged Brain receive PR #178](https://github.com/neomjs/neo-agent-brain/pull/178)
- [Engine deletion neomjs/neo#17791](https://github.com/neomjs/neo/issues/17791)
- [Enforcement custody neomjs/neo#17783](https://github.com/neomjs/neo/issues/17783)
- [Skills consumer PR neomjs/neo#17799](https://github.com/neomjs/neo/pull/17799)

Origin Session ID: `bf696f89-3809-4d19-97e1-08a769ba825f`

Retrieval Hint: `Brain post-merge 21da6802 repin neo.mjs/ai closure ProtoSource pre-push runtime root`



## Timeline

- 2026-08-26T21:14:09Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-26T21:14:10Z @neo-gpt-emmy added the `bug` label
- 2026-08-26T21:14:11Z @neo-gpt-emmy added the `ai` label
- 2026-08-26T21:14:11Z @neo-gpt-emmy added the `regression` label
- 2026-08-26T21:14:11Z @neo-gpt-emmy added the `build` label
- 2026-08-26T21:14:11Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-26T21:44:55Z @tobiu cross-referenced by PR #180
- 2026-08-26T21:56:35Z @neo-gpt cross-referenced by PR #17806
- 2026-08-26T22:24:06Z @tobiu referenced in commit `3817cde` - "feat(brain): receive remaining split assets (#179)"
- 2026-08-26T22:24:06Z @tobiu referenced in commit `1d66b8b` - "fix(brain): support explicit pin-fetch targets (#179)"
- 2026-08-26T22:35:28Z @tobiu referenced in commit `5c8e70a` - "fix(brain): narrow follow-up to verified custody (#179)"
- 2026-08-26T22:37:24Z @tobiu referenced in commit `bd01434` - "ci(brain): retrigger narrowed validation (#179)"
- 2026-08-26T23:08:11Z @tobiu referenced in commit `cd251a9` - "Merge pull request #180 from neomjs/codex/179-engine-ai-closure

feat(brain): complete receive-side closure (#179)"
- 2026-08-26T23:08:11Z @tobiu closed this issue
- 2026-08-27T09:19:59Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-27T09:42:34Z @neo-opus-vega cross-referenced by PR #183
- 2026-08-27T09:43:23Z @neo-opus-vega cross-referenced by PR #185

