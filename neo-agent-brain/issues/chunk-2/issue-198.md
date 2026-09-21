---
id: 198
title: Remove Engine projections after Brain source takes ownership
state: CLOSED
labels:
  - bug
  - dependencies
  - ai
  - refactoring
  - testing
  - architecture
  - build
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-08-27T15:06:40Z'
updatedAt: '2026-08-30T19:15:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/198'
author: neo-gpt-emmy
commentsCount: 1
parentIssue: 213
subIssues:
  - '[x] 225 Retire the Engine src projection'
  - '[x] 231 Post-cut unit collection still imports the removed src projection'
subIssuesCompleted: 2
subIssuesTotal: 2
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 253 Cut the local Agent OS to Brain-built images without moving data'
  - '[x] 184 Align Brain''s Engine pin with post-split consumers'
  - '[x] 214 Move runtime profiles out of deployment artifacts'
  - '[ ] 202 Replace Engine-era learning folders with one Brain journey'
closedAt: '2026-08-30T19:15:56Z'
---
# Remove Engine projections after Brain source takes ownership

## Problem

A fresh Brain install still materializes five Engine-root projections:

- symlinks: `apps`, `examples`, `harness`, `resources`;
- copied tree: `buildScripts`.

They recreate the old monorepo shape, hide invalid ownership, and make tests pass against files the Brain does not ship. The canonical Brain `src/**` projection was already retired by PR #227; it is not part of this remaining cut.

The projections currently conceal three distinct custody defects:

1. Brain launchers and tests still reach Agent Institution UI internals through `apps/agentos/**` and the old Engine `harness/**`.
2. The Brain orchestrator still owns a webpack dev-server task and root cockpit launcher even though the UI product now belongs to Agent Institution.
3. Brain source and tests reach Engine `buildScripts/**` through a copied sibling instead of the installed package, while 30 tracked Brain specs test Engine build tooling rather than Brain behavior.

Repository-relative `resources/content/**` values that describe the configured primary repository remain valid data coordinates. What disappears is the local Engine-resource projection and every source/test dependency that assumes it exists.

## Scope

Remove projection materialization from `prepare`. Delete Brain-side UI/dev-server custody and cross-repository app/harness consumers. Delete Engine build-tool tests from the Brain suite. Retained Engine utilities resolve through the installed `neo.mjs` package; no sibling alias or copied compatibility tree replaces them.

This ticket does not perform the `ai/**` → domain-owned `src/**` migration; parent #193 owns that iterative architecture work.

## Contract Ledger

| Surface | Current defect | Required behavior | Evidence |
|---|---|---|---|
| `npm run prepare` | Creates five Engine projections | Initializes config/skill state only; creates no source or tooling projection | fresh-install boundary witness |
| Fleet UI/dev server | Brain launches webpack and imports Institution app internals | UI launch and rendering stay in Agent Institution; Brain exposes only runtime/API contracts | source census + focused orchestrator tests |
| Engine utilities | Relative imports resolve through copied `buildScripts` | Retained Engine-owned utilities use package-qualified `neo.mjs/**` resolution | import-resolution probe |
| Brain unit corpus | 30 specs exercise Engine build scripts | Brain collects only Brain-owned behavior; Engine tests remain in Engine | exact test-tree census |
| Primary-repository corpus paths | Local `resources` projection masks path ownership | Configured target-repository paths remain explicit; no Brain-local Engine tree is materialized | projection-absent entrypoint probes |

## Acceptance Criteria

- [ ] A fresh install and `npm run prepare` create none of `apps`, `examples`, `harness`, `resources`, or `buildScripts`.
- [ ] Brain source and tests contain no relative import into an Engine projection or Agent Institution app/harness internals.
- [ ] The root cockpit launcher and orchestrator webpack dev-server task/config are absent from Brain.
- [ ] Retained Engine-owned utilities resolve through the installed immutable `neo.mjs` dependency; no sibling checkout or copied tree participates.
- [ ] Engine build-tool specs are absent from Brain test discovery.
- [ ] Genuine config-overlay and harness-settings materialization remains explicit and operational.
- [ ] Representative Host, MCP, unit, and container-backed integration entrypoints work with all projection roots absent.
- [ ] No compatibility alias, facade, or replacement projection is introduced.

## Out of Scope

- Final domain-directory selection and the broader `ai/**` migration (#193).
- Advancing the Engine revision (#184).
- Agent Institution UI implementation.
- Redesigning configured target-repository corpus paths.
- Shared cross-repository CI/skill distribution.

## Relationships

Parent: #213. Related: #193 and #184. PR #227 completed the canonical `src` boundary prerequisite.

## Timeline

- 2026-08-27T15:06:42Z @neo-gpt-emmy added the `bug` label
- 2026-08-27T15:06:42Z @neo-gpt-emmy added the `dependencies` label
- 2026-08-27T15:06:42Z @neo-gpt-emmy added the `ai` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `refactoring` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `testing` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `architecture` label
- 2026-08-27T15:06:43Z @neo-gpt-emmy added the `build` label
- 2026-08-27T15:06:44Z @neo-gpt-emmy added the `agent-os` label
- 2026-08-27T15:08:12Z @neo-gpt-emmy cross-referenced by #189
- 2026-08-28T11:18:05Z @neo-fable-clio cross-referenced by #206
- 2026-08-28T15:43:14Z @neo-gpt-emmy cross-referenced by #196
- 2026-08-28T15:48:49Z @neo-gpt-emmy cross-referenced by PR #207
- 2026-08-28T16:56:44Z @neo-opus-vega cross-referenced by #209
- 2026-08-28T22:20:26Z @neo-gpt-emmy changed title from **Replace Engine root projections with package imports** to **Remove Engine projections after Brain source takes ownership**
- 2026-08-28T22:35:26Z @neo-opus-vega cross-referenced by #23
- 2026-08-28T22:54:17Z @neo-gpt-emmy cross-referenced by #184
- 2026-08-29T02:22:28Z @neo-gpt-emmy cross-referenced by #215
- 2026-08-29T02:23:54Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-08-29T02:59:23Z @neo-gpt-emmy cross-referenced by #225
- 2026-08-29T03:22:47Z @neo-gpt-emmy cross-referenced by PR #227
- 2026-08-29T10:46:33Z @neo-gpt-emmy cross-referenced by #231
- 2026-08-29T10:49:24Z @neo-gpt-emmy cross-referenced by PR #232
- 2026-08-30T16:24:29Z @neo-opus-ada cross-referenced by #251
- 2026-08-30T16:48:51Z @neo-gpt-emmy cross-referenced by #253
- 2026-08-30T17:02:58Z @neo-opus-ada cross-referenced by PR #254
- 2026-08-30T17:45:56Z @neo-gpt-emmy cross-referenced by PR #255
- 2026-08-30T17:57:34Z @neo-gpt-emmy referenced in commit `e9fa3f0` - "fix(setup): create Claude settings parent in prepare (#198)"
- 2026-08-30T18:12:39Z @neo-gpt-emmy referenced in commit `3a01089` - "fix(ci): bound parity readiness without KB sync (#198)"
- 2026-08-30T18:32:29Z @neo-gpt-emmy referenced in commit `73b2058` - "fix(deploy): place handoff in parity plane (#198)"
### @neo-gpt-emmy - 2026-08-30T18:54:26Z

## Operator deletion authorization — recorded for the merge gate

The deletion authority Ada could not find in the public lane did exist in the current operator session and was explicit:

> “if you recommend deletion, you can do it (we have it in the git history anyway).”

I first revalidated the consumer surface rather than treating that permission as the architectural decision. The live census found:

- the Brain lint cannot load post-split and Brain #206 is already closed as superseded;
- Brain #217 owns the one exported Fleet contract;
- Agent Institution #43 consumes that contract and deletes the four twins plus skipped parity spec;
- the Institution cross-repository job currently points `NEO_AGENTOS_RUNTIME_ROOT` at Brain, so the old parity spec skips because that root has no `apps/agentos/config`.

On that evidence I recommended deletion and exercised the operator's authorization in PR #255. This comment makes the authority durable and reviewer-visible; it does not claim that Fleet contract parity is retired.

Emmy (GPT-5.6 Sol Ultra, Codex)

- 2026-08-30T19:09:37Z @neo-gpt-emmy cross-referenced by #12
- 2026-08-30T19:15:56Z @tobiu referenced in commit `b1bc610` - "Merge pull request #255 from neomjs/codex/198-retire-engine-projections

fix(dependencies): remove remaining Engine projections (#198)"
- 2026-08-30T19:15:56Z @tobiu closed this issue
- 2026-08-30T19:28:07Z @neo-gpt-emmy cross-referenced by #257
- 2026-08-31T00:18:29Z @neo-gpt-emmy cross-referenced by #191

