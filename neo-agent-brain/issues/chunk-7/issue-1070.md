---
id: 1070
title: A retained provenance count prevents summary receipt recovery
state: CLOSED
labels:
  - bug
  - ai
  - agent-os
assignees:
  - neo-gpt-euclid
createdAt: '2026-10-16T18:19:36Z'
updatedAt: '2026-10-16T19:15:04Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1070'
author: neo-gpt-euclid
commentsCount: 1
parentIssue: 64
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-16T19:15:04Z'
---
# A retained provenance count prevents summary receipt recovery

## Context

The [observed summary-child failures on #64](https://github.com/neomjs/neo-agent-brain/issues/64#issuecomment-6219476805) repeat before drift work can progress. A targeted read-only comparison on 16 October, serving Brain `d628999a`, identifies the actual mismatch: a completed version-2 receipt and live row have identical documents; the receipt omits `unclassifiedSourceCount`, while the projection has that numeric field. No other owned metadata differs, and the receipt has no null-valued fields. The diagnostic exposed only key presence/types/hashes, not summary text or identity values; no live data was changed.

Vega offered the bounded repair after an isolated control; Euclid accepted. This is a source-repair leaf under #64, not a transport-regression or server-restart claim.

## The Problem

`SessionService.resolveSummarySourceProvenance()` starts its count at zero, but synthesis writes the field only when positive. Chroma upsert merges metadata, so an extra prior count survives replay with an omitted key. Version-2 recovery attests absence, upserts the same omission, then fails verification again.

An isolated in-memory SQLite/Chroma-shaped control reproduces that precise mechanism twice: the projection retains a synthetic count of 7, each pass throws the receipt-verification error, both jobs remain unacknowledged, and envelope bytes remain unchanged. The existing ownership-absence refusal test also passes; that guard must stay strict.

## The Architectural Reality

At current Brain `924b8bdd7afbdf0f5e3db5f8a2ae628b86abf67e`:

- [Producer count semantics](https://github.com/neomjs/neo-agent-brain/blob/924b8bdd7afbdf0f5e3db5f8a2ae628b86abf67e/ai/services/memory-core/SessionService.mjs#L118) establish omission as zero; [conditional publication](https://github.com/neomjs/neo-agent-brain/blob/924b8bdd7afbdf0f5e3db5f8a2ae628b86abf67e/ai/services/memory-core/SessionService.mjs#L804) creates the missing-key shape.
- [Receipt matching](https://github.com/neomjs/neo-agent-brain/blob/924b8bdd7afbdf0f5e3db5f8a2ae628b86abf67e/ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs#L288) and [replay](https://github.com/neomjs/neo-agent-brain/blob/924b8bdd7afbdf0f5e3db5f8a2ae628b86abf67e/ai/services/memory-core/helpers/sessionSummaryReceiptStore.mjs#L449) own convergence. Recovery precedes synthesis-failure backoff.
- Design authority: the helper's JSDoc requires replay of synthesis-owned metadata while preserving downstream Dream overlays; the producer computes a numeric zero for fully classified sources.

## The Fix

Publish `unclassifiedSourceCount: 0` explicitly for new synthesis. For **version 2 with this key absent only**, construct replay metadata with numeric zero, without changing the decoded receipt or stored envelope. Treat existing projection absence or numeric zero as equivalent to that historical omission, avoiding needless replay of already-converged rows. Explicit zero/positive receipts remain strict; version 1 retains its frozen key set; `userId` absence remains strict.

Use the existing service/helper and owning specs; no new module, daemon, schema or config leaf.

### Contract Ledger

| Target surface | Source of authority | Behavior | Fallback | Docs | Evidence |
| --- | --- | --- | --- | --- | --- |
| New summary metadata | `resolveSummarySourceProvenance` | Publish the computed numeric count, including zero, to both projection and receipt | No fabricated owner or source identity | Producer JSDoc | Classified-source producer control |
| v2 omitted count | Existing receipt version and producer omission rule | Absence/zero match; stale residue is replaced with zero through a separate replay object | No coercion of string/Boolean/null to zero | Receipt-helper JSDoc | Red→green legacy-count controls; real SDK fixture |
| Other receipt fields | Versioned owned-key contract | Retain strict explicit values, v1 keys, documents, owner absence and Dream overlays | Existing ownership/malformed/transport refusals remain | Existing helper contract | Negative and preservation controls |

Decision Record impact: none; this restores the existing provenance count's representation without changing authority.

## Acceptance Criteria

- [ ] AC-1: New synthesis with classified sources writes numeric zero to the projection and staged receipt; positive counts remain exact.
- [ ] AC-2: An omitted-count v2 receipt over a stale positive count replays once, acknowledges the healthy sibling, and performs no second-pass upsert. Receipt bytes remain identical.
- [ ] AC-3: Omitted-count v2 receipts over absent/zero projections are no-ops. Wrong types and explicit-count changes still require repair; no truthiness/coercion match.
- [ ] AC-4: v1 frozen-key behavior, strict `userId` absence refusal, document equality and Dream-owned overlays remain intact.
- [ ] AC-5: The owning disposable-Chroma/real SDK harness proves count replacement and second-pass convergence with no shared collection or provider mutation.

## Out of Scope

Live-plane data repair, deletion, re-keying, restart or adoption; automatic write replay; generic poison-receipt quarantine/backoff; null-value normalization; HTTP closure causes. The next deployed observation remains with #64 after the normal source adoption. A source merge is not a live repair receipt.

## Avoided Traps

Suppressing verification or adding backoff alone leaves the known mismatch. Deleting `userId` or treating every optional key as absent would weaken ownership. Rewriting historical envelopes loses their exact retained input; replay derives only this version-bound numeric representation.

## Related

Parent: #64. Adjacent closed synthesis-backoff history: #438. Transport/adoption context: #1062; no inferred causal edge.

Origin Session ID: 5333f9c4-02c5-43fb-b40c-75216654a278
Retrieval Hint: `sessionSummaryReceiptStore unclassifiedSourceCount absent v2 merging upsert replay verification`

Creation checks: latest 20 open Brain issues and recent all-state A2A claims rechecked immediately before creation; no equivalent or competing claim. Six own open assignments were inspected; none owns this repair. Native MC/KB symptom queries returned unrelated results, not evidence of absence; live source and isolated controls establish the problem. Structure map ran successfully: existing `ai/services/memory-core/` and `helpers/` own the change.

## Timeline

- 2026-10-16T18:19:37Z @neo-gpt-euclid assigned to @neo-gpt-euclid
- 2026-10-16T18:19:39Z @neo-gpt-euclid added the `bug` label
- 2026-10-16T18:19:39Z @neo-gpt-euclid added the `ai` label
- 2026-10-16T18:19:40Z @neo-gpt-euclid added the `agent-os` label
- 2026-10-16T18:19:40Z @neo-gpt-euclid added parent issue #64
### @neo-gpt-euclid - 2026-10-16T18:21:06Z

[ARCH_ALIGNMENT] Prescription checked: `ai/services/memory-core/SessionService.mjs` owns count issuance; `helpers/sessionSummaryReceiptStore.mjs` owns versioned projection/replay convergence. A scheduler/config/backoff change would mask this deterministic metadata mismatch. The same-session ticket has an independent bounded source read plus the exact read-only field comparison and isolated reproduction. Keep v2 omitted-count normalization separate from `userId`, explicit counts and v1 frozen keys. Self-assignment was atomic at creation; the subsequent guarded add correctly reported the existing `neo-gpt` assignee. Native pre-brief is not yet ingested (`NODE_NOT_FOUND`); live GitHub/source are the intake authority.

Origin Session ID: 5333f9c4-02c5-43fb-b40c-75216654a278

- 2026-10-16T18:33:14Z @neo-gpt-euclid cross-referenced by PR #1071
- 2026-10-16T19:15:04Z @tobiu referenced in commit `fa34303` - "fix(memory-core): converge retained zero provenance counts (#1070) (#1071)"
- 2026-10-16T19:15:04Z @tobiu closed this issue
- 2026-10-16T19:18:55Z @neo-gpt-euclid cross-referenced by #64

