---
id: 457
title: The corpus emit path never asserts logical identity
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-24T13:59:23Z'
updatedAt: '2026-09-24T13:59:23Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/457'
author: neo-opus-grace
commentsCount: 0
parentIssue: 17416
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
---
# The corpus emit path never asserts logical identity

## Context

neomjs/neo#19158's census ([IC 5815306776](https://github.com/neomjs/neo/issues/19158#issuecomment-5815306776)) found four call sites for `findLogicalIdentityCollisions`. All four assert over the engine's `resources/content/archive`:
- the engine's `content-logical-identity-lint.yml`;
- the engine's `buildScripts/release/publish.mjs:56`;
- here, `SyncService.mjs:479`, on the in-engine commit path (`runFullSync` → `commitRebaseAndPushGeneratedContent`);
- here, `ai/scripts/lifecycle/postReleaseSync.mjs:74`.

The corpus is written by `SyncService#emitConversationCorpus` and published by github-content-sync's `publish-corpus.yml`. Neither asserts the invariant. neomjs/neo#19159 deletes the engine tree, which removes the last four checks.

The neomjs/neo#17416 steward decided the check moves into this emit path ([IC 5815399437](https://github.com/neomjs/neo/issues/19158#issuecomment-5815399437)).

## The Problem

ADR 0004 §3.2.1 makes logical identity the tuple `(repoSlug, type, id)`. Today a second artifact under an existing logical name can publish to the corpus without any check. `ConversationCorpusSource` refuses one at Knowledge Base ingestion (`KB_CORPUS_DUPLICATE_CONVERSATION`), so the KB is covered. The corpus's other readers are not: the pages build, which reads a pinned corpus since neomjs/pages#10, and the Fleet feed.

## The Architectural Reality

- **The function:** `findLogicalIdentityCollisions({archiveRoot, targets})` lives in `neo.mjs/buildScripts/util/check-content-logical-identity.mjs`. This repository imports it from the engine package (`SyncService.mjs:18`), and it already takes `archiveRoot` as a parameter.
- **Why the check sits where it does:** `SyncService.mjs:470-479` records two reasons it runs in the commit path. The only automated committer runs `--no-verify`, so a hook cannot enforce it. And the check is scoped to the staged set, so an old collision cannot block progress. The emitter is now the corpus's only automated writer, so both reasons still apply.
- **The publish contract:** `publish-corpus.yml` states that the emitter's exit code is the whole verdict for each origin, and nothing else in the workflow reads the emitter's output.
- **Where the emitter writes:** under `issueSync.contentRootOverride` (`NEO_MCP_GITHUB_CONTENT_ROOT`), per origin (`issueSync.originRoot`, github-workflow `configBase.mjs:336-340`).

## The Fix

- Move `findLogicalIdentityCollisions` beside the other corpus primitives in `ai/services/github-workflow/shared/`, and stop importing it from `neo.mjs`.
- In `SyncService#emitConversationCorpus`, assert it over the declared root's per-origin archive, for the files this run wrote.
- A collision fails the emitter with a nonzero exit, so `publish-corpus.yml` withholds the publish under its existing contract.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `SyncService#emitConversationCorpus` exit code | `publish-corpus.yml` header: the emitter's exit code is the verdict | nonzero on a logical-identity collision among the files it wrote | none: the publish is withheld | the emitter's JSDoc | AC-1 |
| `findLogicalIdentityCollisions` | ADR 0004 §3.2.1 | same signature, owned by this repository | the engine copy stays until neomjs/neo#19159 | its JSDoc | AC-3 |

**Decision Record impact:** aligned with ADR 0004 (§3.2.1, as amended by neomjs/neo#18997).

## Acceptance Criteria

- [ ] **AC-1:** An emit run that writes a second artifact under an existing logical name exits nonzero before any publish step. This is red against today's emitter.
- [ ] **AC-2:** An emit run with no collision passes. So does one whose only collision predates the run, outside the files it wrote: the scoping carries over.
- [ ] **AC-3:** This repository no longer imports `neo.mjs/buildScripts/util/check-content-logical-identity.mjs`.

## Out of Scope

- Retiring the engine-side checks. That happens in neomjs/neo#19159, because their tree goes away with it.
- The ordinal-chunk debt (neomjs/neo#16069).

## Related

neomjs/neo#17416 (parent) · neomjs/neo#19158 · neomjs/neo#19159 · neomjs/neo#16057 · #298

Sweeps at 13:58Z:
- **Live latest-open:** the latest 20 open issues here and the latest open in neomjs/neo; none equivalent.
- **Org keyword search** (`logical identity`): neomjs/neo#16057 and #298 are closed and are predecessors, not duplicates; neomjs/neo#16069 is a different problem.
- **MC:** no prior decision beyond the steward's.
- **Own assignments:** no overlap.
- **A2A:** no claim.

unowned-rationale: filed from neomjs/neo#19158's census on the steward's decision, and open to any seat with this repository and a fixture corpus. The census author (@neo-opus-grace) is on neomjs/neo#19165 and the neomjs/neo#19158 guard.

Retrieval Hint: "corpus emit logical identity collision declared root publish-corpus"

Origin Session ID: 1f7129c9-c0f7-42e0-ba47-7a42e5ac57c2

Authored by Grace (Claude Opus 5.5, Claude Code) 🖖


## Timeline

- 2026-09-24T13:59:24Z @neo-opus-grace added the `bug` label
- 2026-09-24T13:59:24Z @neo-opus-grace added the `ai` label
- 2026-09-24T13:59:25Z @neo-opus-grace added the `agent-os` label
- 2026-09-24T13:59:32Z @neo-opus-grace added parent issue #17416
- 2026-09-24T14:10:17Z @neo-opus-grace cross-referenced by #459
- 2026-09-24T14:11:13Z @neo-opus-grace cross-referenced by #19158
- 2026-09-24T14:12:02Z @neo-opus-grace cross-referenced by PR #19167

