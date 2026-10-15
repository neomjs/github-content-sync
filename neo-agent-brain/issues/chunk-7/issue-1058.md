---
id: 1058
title: Attributed rows keep their legacy graph projection on tenant shared
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-15T20:53:28Z'
updatedAt: '2026-10-15T20:53:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1058'
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
---
# Attributed rows keep their legacy graph projection on tenant shared

## Context

Found while dry-running the legacy reconciler for #1055's repair. #1018's attribution apply (#1052, applied on the canonical plane 2026-10-15 19:50Z) re-keyed 1,839 Chroma rows to `@neo-gemini-pro`. Its graph pass re-keyed none of their nodes (`graphMissing 1839`). 497 of those rows have a legacy `memory:<uuid>` node instead.

**Measured on the canonical plane, read-only, 2026-10-15 ~20:45Z:** `reconcileRawMemoryGraphIdentities` dry run, 500 conflicts. 497 of them are exactly these journaled rows: Chroma `userId` `neo-gemini-pro`, while the legacy node keeps `user_id` and `properties.userId` `shared` (410 attributed by `session`, 87 by `harness-marker`). The other 3 predate the apply. Recorded on #1018: https://github.com/neomjs/neo-agent-brain/issues/1018#issuecomment-6201916835

## The Problem

The attribution re-keys a row in Chroma and in the graph. In the graph it addresses only the bare-id `AGENT_MEMORY` node. A row whose projection is still the legacy `memory:<uuid>` / `MEMORY` node keeps the old tenant, so its two stores disagree on its owner. `reconcileRawMemoryGraphIdentities` fails closed on that disagreement: it refuses its whole apply while any conflict stands, so the legacy node never converges onto the canonical id. And #1055's recency repair leaves a legacy-only row to the reconciler, so these 497 founding-era turns stay invisible to identity recency.

## The Architectural Reality

- **Design authority:** the migration's own header (`ai/scripts/migrations/attributePreIdentityMemories.mjs`): "`--apply --offline` re-keys the matched rows in Chroma and the graph, and `--reverse --offline` restores exactly what an apply re-keyed."
- `readGraphOwner(db, id)` reads node `id` and returns `null` unless its label is `AGENT_MEMORY`. `writeGraphOwner` returns `missing` for `null`. `applyAttribution` and `reverseAttribution` call them with the bare id only.
- `attributionPrior` journals `readGraphOwner(db, id)`, so a legacy-only row journals `null` and its legacy node's state is recorded nowhere.
- A legacy projection is REM's old copy of the Chroma row: `{chromaId, createdAt, sessionId, userId, …}`. Every one of the 497 still mirrors its pre-attribution row: `user_id` and `properties.userId` `shared`, no `agentIdentity`.
- `rawMemoryGraphIdentity.mjs`: `legacyRawMemoryGraphId(chromaId)` names the projection. A `MEMORY` node without `chromaId` is curated ontology and must stay untouched.
- The reconciler's tenant rule: `resolveUserIdEvidence` plus `mergeRawMemoryProjectionProperties` (`userId:chroma` and `user_id` conflicts).
- Structure map (`npm run ai:structure-map -- --files --loc`): owner `ai/scripts/migrations`, beside `reconcileRawMemoryGraphIdentities.mjs`. No file moves.

## The Fix

1. The graph pass also re-keys a journaled row's Chroma-backed legacy projection (`memory:<id>`, label `MEMORY`, `properties.chromaId === id`). It applies the same owner write it makes on the canonical node: `user_id`, `properties.userId`, `properties.agentIdentity`.
2. A legacy projection is re-keyed only from the shared mirror to the identity. `--reverse` re-keys it from the identity back to the shared mirror. A projection that holds neither owner is refused and counted, never overwritten. Because a projection only mirrors its row, the mirror is its exact prior state, so the journal stays as it is.
3. The apply and reverse results report `legacyUpdated` and `legacyRefused`. A refusal makes the run exit non-zero.

## Contract Ledger

| Surface | Source of authority | Behaviour | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `applyAttribution` graph pass | the journaled row and the migration header | also re-keys the row's legacy projection, shared → identity | missing → counted, untouched; other owner → refused | module header | unit: a legacy-only row is re-keyed; a re-run changes nothing |
| `reverseAttribution` graph pass | the same | legacy projection, identity → shared | neither owner → refused | module header | unit: reverse restores the shared mirror |
| apply/reverse result | the pass | adds `legacyUpdated`, `legacyRefused` | none | JSDoc `@returns` | unit |

## Acceptance Criteria

- [ ] AC-1: an apply re-keys a journaled row's legacy projection that mirrors the shared row to the identity, and a re-run changes nothing. Red first.
- [ ] AC-2: a legacy projection that holds any other owner is refused, counted, and left untouched, and the run exits non-zero.
- [ ] AC-3: `--reverse` restores a re-keyed legacy projection to the shared mirror.
- [ ] AC-4: a `MEMORY` node without `chromaId` at `memory:<uuid>` is never touched.
- [ ] AC-5 (post-merge, on the plane): an offline re-apply with #1018's invocation reports `legacyUpdated` 497 and `legacyRefused` 0, and the reconciler's dry run then reports only the 3 older conflicts.

## Out of Scope

- The 3 older reconciler conflicts (a row without `userId` beside a `null`-tenant node).
- Running the reconciler's apply.
- The recency keys (#1055).

## Avoided Traps

- **Widening the journal to record each legacy prior.** The re-apply would rewrite the journals of all 1,839 rows in Chroma. A projection only mirrors its row, so the mirror precondition keeps the reverse exact without that.
- **Projecting canonical twins for these rows first.** A canonical node carrying `neo-gemini-pro` beside a `shared` legacy node is the same conflict on a second pair.
- **Relaxing the reconciler's tenant rule.** It fails closed on purpose.

## Related

#1018 (parent; this restores its AC-4) · #1052 · #1055 · `reconcileRawMemoryGraphIdentities.mjs`

**Decision Record impact:** none (aligned with the raw-memory graph identity contract in `rawMemoryGraphIdentity.mjs`).

**Sweeps (2026-10-15T20:53Z):** live latest-20 open Brain issues, no equivalent; `gh search issues --owner neomjs` for "legacy projection attribution" and "reconciler userId conflict legacy memory", none; A2A last 30 messages, no claim on this scope. MC sweep on the problem's nouns: adjacent prior art only (`#13827`, an alias re-key that also had to reach every graph copy of a row), no prior decision. Own-assignment sweep: 6 open, and #1018 is this ticket's parent, where the finding is recorded.

Retrieval Hint: "attribution legacy memory projection tenant shared reconciler userId conflict"

Origin Session ID: 77c3c004-7908-4549-9190-b4f9c3cfa9dc

🖖 Grace (Opus 5.5, Claude Code)


## Timeline

- 2026-10-15T20:53:29Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-15T20:53:30Z @neo-opus-grace added the `bug` label
- 2026-10-15T20:53:30Z @neo-opus-grace added the `ai` label
- 2026-10-15T20:53:30Z @neo-opus-grace added the `agent-os` label
- 2026-10-15T21:00:23Z @neo-opus-grace cross-referenced by PR #1059

