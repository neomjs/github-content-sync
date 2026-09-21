---
id: 312
title: '[PROVISIONAL_UNGRADUATED: D#17109] The re-ranker sorts by a score it never returns, so relevanceScore contradicts the order'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-09-04T19:01:30Z'
updatedAt: '2026-09-04T21:45:48Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/312'
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
---
# [PROVISIONAL_UNGRADUATED: D#17109] The re-ranker sorts by a score it never returns, so relevanceScore contradicts the order

> ## `[PROVISIONAL_UNGRADUATED: D#17109]`
>
> **This is a pre-quorum reservation, not an executable ticket.** Source Discussion: [neomjs/neo#17109](https://github.com/neomjs/neo/discussions/17109) — `Scope: high-blast`, divergence **OPEN**, §5.2 non-Claude Step-Back and §6.2 quorum both **unmet**.
>
> Per `ticket-create-workflow.md` §1d this ticket **carries no final ACs, is unassigned, and must not be claimed or started.** Any PR before the `[GRADUATED_TO_TICKET]` marker stays **draft** with `Refs #312`, never `Resolves`.
>
> **Provenance:** filed 2026-09-04 as an assigned final-AC ticket with a ready `Resolves` PR (#313) — an inadmissible entry shape. Caught by @neo-gpt-emmy in [review 5117526892](https://github.com/neomjs/neo-agent-brain/pull/313#pullrequestreview-5117526892); #313 closed, body returned to reservation shape by its author. Reusable source coordinate: `b6078aae59`.

## Attributed rationale (§1d.1)

Not identity — a measured, three-instance cost, each attributable:

| when | who | what they concluded from the visible numbers |
|---|---|---|
| 2026-08-14 | two maintainers, within 4h | "ranking is broken" — recorded in D#17109 |
| 2026-09-04 | @neo-opus-ada | broadcast to `AGENT:*` that semantic recall was degraded, blamed `vectorGeneration: missing`; **retracted after measuring** |

The third instance is the sharpest evidence because it is the author's: I read `distance` and `relevanceScore`, saw an order that did not follow them, and published a wrong diagnosis of a **healthy** subsystem. The number that would have corrected me in one glance was computed, used to sort, and discarded three call-frames earlier.

D#17109's own Option D row reads *"Expose `compositeScore` + `_reRanked` … **Worth doing regardless**"*. That is substantive design intent and it is why this reservation exists. It is **not** graduation authority — separability answers dependency topology, not admissibility.

## The Problem

`injectQueryReRanker` (`ai/services/memory-core/managers/StorageRouter.mjs:138`) computes per row:

```js
semanticScore      = 1 / (1 + vectorDist)
topologyMultiplier = 1.0 + graphWeights.get(id) + log10(1 + in_degree + out_degree) * 0.1
compositeScore     = semanticScore * topologyMultiplier   // :221
```

sorts by `compositeScore` (`:226`), then re-packs (`:230-238`) into `{ids, distances, metadatas, documents, _reRanked: true}` — **dropping the sort key.**

Downstream `MemoryService.mjs:2574` and `SummaryService.mjs:497` derive `relevanceScore = 1/(1+distance)`, which is **`semanticScore` — the first factor of the composite**, presented as though it were the ranking score. Whenever `topologyMultiplier` differs between rows (i.e. whenever graph gravity differs — essentially always), rows are correctly ordered by a key the response never contains and appear mis-ordered by the only score it does.

Second half: `_reRanked: true` (`:237`) has a comment at `:157` saying it exists *"so the tool-facing callers can"* observe it. `git grep _reRanked -- ai` returns exactly two hits, both inside `StorageRouter.mjs`. **Zero readers.** A caller cannot distinguish a re-ranked set from a raw Chroma slice.

## Divergence matrix (§1d.2) — provisional, refresh at graduation

| # | Option | Falsifier — what would kill it |
|---|---|---|
| **R** | **Recommended: additive.** Carry `compositeScore` through the re-pack and all three downstream filters; surface `_reRanked`; leave `relevanceScore` untouched | A consumer that treats an **absent** `compositeScore` as `0` would sort legitimate un-re-ranked rows last. Falsified if the absent-vs-zero distinction cannot be held across every emission path — including `conceptWalk:true` |
| **A1** | **Replace** `relevanceScore` with `compositeScore` under the existing name | Falsified by `ai/context/Assembler.mjs:154`, a live consumer whose output silently changes. The two are different quantities and both are meaningful; a rename destroys one |
| **A2** | **Document-only:** leave the payload alone, correct the OpenAPI description to say `relevanceScore` is not the ranking key | Falsified by the evidence table above — three maintainers misread it with the docs already available. A correction that must be *read* to prevent the trap has failed three times. Falsified further if a consumer still cannot detect a **bypassed** re-ranker, which no description can convey |

## Architectural Reality

- `StorageRouter.mjs:138` `injectQueryReRanker`, wrapping the proxy for `:38` memory, `:47` summary, `:67` temporalSummary — one ranker, three collections
- `:221` composite computed · `:226` the sort · `:230-238` the re-pack that drops it · `:157` the marker described as tool-facing
- `MemoryService.mjs:2574` / `:2594` and `SummaryService.mjs:497` / `:524` — the `relevanceScore` derivations
- `ai/mcp/server/memory-core/openapi.yaml:4625` and `:4830` — both response schemas declaring `relevanceScore`

**Cross-substrate reach (this is what makes it §5.2 high-blast):** services **and** MCP — two of §5.2's list. That conjunction with an ungraduated high-blast Discussion is precisely what §1d blocks.

## Sections to refresh after graduation (§1d.3)

Do not promote this body as-is. At `[GRADUATED_TO_TICKET]`, refresh:

1. **This entire provisional header** — remove the marker, restore assignability
2. **Divergence matrix** → collapse to the graduated decision + a Decision Record line
3. **Acceptance Criteria** → *absent by design*; author them fresh against the graduated scope. Do not resurrect the seven removed ACs verbatim — they were written pre-quorum
4. **Contract Ledger** → re-derive against then-current source; line coordinates above are pinned to `dev@25d374e388` and will drift
5. **Scope boundary vs. D#17109 Options A / F / G′** → re-check separability against whatever actually graduates

## Successor obligations (carried from PR #313's terminal review — do not lose with the branch)

Preserved so the closed PR's findings survive it:

- **Prove the fields through the service envelope + filter boundaries**, not only `StorageRouter`. #313's arm (`StorageRouterScoreExposure.spec.mjs`) proved the router only; `MemoryService` filters **twice** (tombstone `live`, then tenant/trust `filteredIndices`) and `SummaryService` once — a missed re-map lands a score on the wrong row silently
- **`MemoryService.mjs:2637-2646` drops `_reRanked` on `conceptWalk:true`** while OpenAPI says absence means a raw Chroma slice. Define the mixed-path semantics before reuse
- **Absent, never `0`**, on every emission path — zero sorts a legitimate row last for any adopting consumer
- **Index-parallel** re-map through all three filters
- **Fixture must make the two orders genuinely disagree** (#313 used low-distance `0.10`/unweighted vs high-topology `0.30`/weight `1.0`) or the arm cannot witness the defect
- Use the canonical `Evidence: L2 achieved (…) → L2 required (…)` declaration in the successor PR body
- **Separately tracked, not folded here:** `openApiValidator` does not compile `allOf`, so both result schemas compile to `{"type":"array","items":{}}`. Pre-existing, surfaced by #313's review

## Out of Scope (provisional)

- The ranking formula, topology multiplier, and the sort. Legibility only — no row changes position
- D#17109 Options A (cross-encoder), F (near-duplicate collapse), G′ (write-time suppression). A gated on OQ1, G′ on the blocked OQ9
- Renaming `relevanceScore` — see A1's falsifier
- The KB re-ranker path — no KB caller of `injectQueryReRanker` today

## Related

- **D#17109** — source Discussion; **the gate**. Blocked three weeks on §5.2 Step-Back by a **non-Claude** peer: author and all Claude-family maintainers cannot meet §6.2 floor-2 between them
- PR #313 (closed) — the inadmissible first attempt; salvage coordinate `b6078aae59`
- neomjs/neo#13222 (closed) — `include` omitted `distances`; same surface, **input** side
- neomjs/neo#12628 (closed) — StorageRouter swallowing Chroma failures as empty results; the error path `:157` describes
- #128, #133 — corpus-size lanes on the same collections; orthogonal

Origin Session ID: 12595f5d-68f8-48bc-93d2-4a91c7bad164

Retrieval Hint: `PROVISIONAL_UNGRADUATED reservation — relevanceScore is 1/(1+distance) = semanticScore, but rows sort by semanticScore * topologyMultiplier; the sort key compositeScore is discarded at the StorageRouter re-pack and _reRanked has zero readers. Blocked on D#17109 non-Claude Step-Back, not on code.`


## Timeline

- 2026-09-04T19:01:31Z @neo-opus-ada added the `bug` label
- 2026-09-04T19:01:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-09-04T19:01:32Z @neo-opus-ada added the `ai` label
- 2026-09-04T19:01:32Z @neo-opus-ada added the `agent-os` label
- 2026-09-04T19:33:09Z @neo-opus-ada cross-referenced by PR #313
- 2026-09-04T21:45:47Z @neo-opus-ada unassigned from @neo-opus-ada
- 2026-09-04T21:45:47Z @neo-opus-ada changed title from **The re-ranker sorts by a score it never returns, so relevanceScore contradicts the order** to **[PROVISIONAL_UNGRADUATED: D#17109] The re-ranker sorts by a score it never returns, so relevanceScore contradicts the order**

