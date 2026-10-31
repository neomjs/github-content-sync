---
id: 1389
title: 'The Golden Path ranks on total inbound weight, so message records never fade'
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-gemini-julia
createdAt: '2026-10-31T00:20:47Z'
updatedAt: '2026-10-31T15:02:44Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1389'
author: neo-opus-vega
commentsCount: 2
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
closedAt: '2026-10-31T15:02:44Z'
---
# The Golden Path ranks on total inbound weight, so message records never fade

## Context

The Computed Golden Path is the plane's steering surface: `get_computed_route` and the handoff's "Computed Golden Path" section tell a booting seat what to pick up. On 2026-10-30 the operator read the route and saw its newest item was neo #17844, open with 95 comments and quiet since 2026-10-12, while neo's ids had passed 20,180; the ticket reads as long done. Captured the same hour as a defect-note (MESSAGE:3598e8bb), promoted here after the evidence below.

Observation and inference are kept apart: the route's contents, the graph's edges and the code paths are observed; the cause is the code path that consumes them, read at `dev` 4ca0f506 (the running plane's revision).

## The Problem

**The route at 22:22Z (the last pass that produced one), as rendered in `sandman_handoff.md`, against the live tracker at 00:0xZ:**

| Rank | Item | Score (semantic / structural) | Live state | Last activity |
|---|---|---|---|---|
| 1 | neo #17844 | 156.50 (0.91 / 154.68) | OPEN, Euclid | 2026-10-12 (19 days) |
| 2 | neo #9849 | 11.85 (0.92 / 10.00) | OPEN, unassigned, created 04-10 | 2026-09-15 |
| 3 | neo #19482 | 7.96 (0.98 / 6.00) | **CLOSED (completed) 2026-10-09** | none |
| 4 | neo #9853 | 7.04 (1.02 / 5.00) | OPEN, created 04-10 | 2026-10-05 |
| 5 | neo #12964 | 6.90 (0.95 / 5.00) | OPEN, Clio | 2026-09-24 |
| 6 | neo #16217 | 5.00 (1.00 / 3.00) | OPEN, Ada | 2026-09-01 |
| 7 | neo #13144 | 4.82 (0.91 / 3.00) | OPEN | 2026-10-29 |
| 8 | neo D#15812 | 4.00 (0.91 / 2.18) | open | 2026-07-24 |
| 9 | neo #14788 | 3.73 (0.92 / 1.89) | OPEN, "by ~July 10" | 2026-10-05 |
| 10 | neo D#10546 | 1.88 (0.94 / 0.00) | open | 2026-05-01 |

Seven of ten are silent past the 14-day threshold the synthesizer already carries (`goldenPathSilentThreadThresholdMs`), one is closed, and the leader outscores the runner-up 15×, entirely on the structural term.

**Where #17844's 154.68 comes from** (`get_neighbors('issue-17844')`, counted): 141 inbound `REFERENCES_TICKET` edges whose sources are `MESSAGE:` nodes, one `GUIDES` edge from `frontier`, nine `DISCUSSED_IN` edges from session summaries. The 141 are A2A message records: every message that ever named the ticket during its Engine lane in September and October, weight 1 each. By `GraphService` design those edges are the sender's record: `decayGlobalTopology` never decays or prunes them, and `getInboundStructuralSupport` counts them in `totalWeight` and excludes them from `decayingWeight` "so archaeology or a prior route cannot masquerade as current swarm motion" (`ai/services/memory-core/GraphService.mjs`, the `isMessageRecordEdge` contract). The ranking then consumes the wrong one of the two.

## The Architectural Reality

- `GoldenPathSynthesizer.mjs` scoring loop: `rawStructScore = structuralSupport.totalWeight` (L1383), `priority = semantic × 2 + struct × 1` (L1407). `decayingWeight` is consumed only by `evaluateDiscussionLiveness` (L1372), the v13.2 Discussion-liveness contract from neo #15105; `GraphService.getInboundStructuralSupport` documents `totalWeight` as "preserves existing structural scoring".
- The silent-thread threshold (14 d) and the stale-assignment threshold (7 d) feed visibility sections only (`buildSilentThreadCandidates`, L1918–1929, via `issueFocusSections.mjs`); the route ignores them.
- `isActionableComputedRecommendation` (`computedGoldenPathRouting.mjs`) gates on node type, id prefix, fixture flag and exclusion labels; state is filtered earlier by the SQLite hydration (`json_extract(n.data, '$.properties.state') = 'OPEN'`), so a closed issue is routed exactly when the graph projection still says OPEN. `get_node('issue-19482')` returns `state: OPEN` tonight while GitHub closed it on 2026-10-09; `issue-19565` (closed 2026-10-10) is CLOSED in the graph, so propagation works in general and #19482 is a node that missed its update. That is a projection defect, captured separately as a defect-note, not fixed here.
- Prior art: neo #13750 (June, PRIO-ZERO "Golden Path frozen 18 days / meta-heavy ranking") was closed by recency-sorting the frontier baseline (#13801), release-focus steering (#13759), the ISSUE+DISCUSSION pool (#13837) and hourly freshness (#13783); the structural term was left as `totalWeight`. neo D#14422 OQ4 (July) named structural cold-start and routed novelty remedies to the Golden Path v2 epic (neomjs/neo-agent-brain#122) as matrix material. Neither touched the message-record mass, which did not exist at that scale until the A2A graph projection (`MESSAGE` nodes with `REFERENCES_TICKET`) made every lane's traffic an inbound edge on its ticket. The route-attribution ledger (#14454 → `routeAttributionLedgerStore.mjs`, `learn/agentos/measurements/golden-path-route-attribution-2026-07-02.md`) is the instrument that shows per-item components in one pass.
- The 23:22Z pass published the `publishUnavailableComputedRoute` shape (status degraded, freshness unverifiable, `kind: none`, zero scored sources) two minutes after the orchestrator's 23:20Z restart, with sibling services still booting. The sidecar carries no reason code (`routeFailure: {status: 'failed'}` is passed without one), and neither the three containers' logs for the last three hours nor the orchestrator's daily log for 2026-10-30 holds a line naming the synthesizer, although the pass wrote at 22:22Z and 23:22Z; at 00:19Z the 23:22Z sidecar was still the current one, three minutes before its expiry. Captured as a defect-note; not this ticket's fix.

Structure map (`npm run ai:structure-map -- --files --loc`): the owning folder is `ai/services/graph/` (`GoldenPathSynthesizer.mjs`, `computedGoldenPathRouting.mjs`, `issueFocusSections.mjs`, `routeAttributionLedgerStore.mjs`); the support projection lives in `ai/services/memory-core/GraphService.mjs`. No new file.

## The Fix

Rank by the projection the engine already computes for exactly this purpose:

1. `GoldenPathSynthesizer.mjs` L1383: `rawStructScore = structuralSupport.decayingWeight` for ISSUE and DISCUSSION candidates. `totalWeight` stays available for the visibility sections and for cold-start parent inheritance (L1388–1399 reads the parent's support; use the same projection there).
2. Update the `getInboundStructuralSupport` JSDoc so `totalWeight` no longer claims to be the ranking input, and the ranking line names the record-edge exclusion as the reason.
3. Evidence in the PR: one route-attribution ledger run on the same candidate pool before and after, listing for each of the top 10 the two projections, the message-record edge count and the latest activity date; the operator's specimen (#17844) must leave the top 10 unless live traffic puts it back.

No new weight, decay factor, threshold or schema. Cold-start (new tickets at ≈0 structural weight) is unchanged and stays with neomjs/neo-agent-brain#122.

## Decision Record impact

aligned-with `adr:0024-native-edge-graph-model` (§2.3: scent decays, facts persist; ADR 0023's invariant). The change consumes the decaying projection the ADR defines; it adds no edge class and changes no protection. No amendment.

## Acceptance Criteria

- [ ] The scoring loop ranks ISSUE and DISCUSSION candidates by `decayingWeight`; message-record edges, protected fact edges and the frontier's own `GUIDES` output no longer contribute to the route order.
- [ ] A route-attribution ledger run on the live pool shows, per routed item, `totalWeight`, `decayingWeight`, the count of `MESSAGE:`-sourced inbound edges and the latest activity date, before and after; committed under `learn/agentos/measurements/` beside the 2026-07-02 ledger.
- [ ] On that run, no item silent longer than `goldenPathSilentThreadThresholdMs` outranks an item with activity inside it on the structural term alone, unless that term is explained by decaying edges in the ledger; the leader's structural term is no longer dominated by message records (state the ratio). *(The "unless" clause was added 2026-10-31 by the author, in the review of PR #1393. GitHub silence and swarm scent are two different activity measures, and this AC meant the scent, as AC-5 already says. How far that scent can be trusted is #1395.)*
- [ ] Existing unit coverage of the scoring loop and the liveness decision passes; a new case feeds one candidate 100 message-record edges and one candidate 3 decaying edges and asserts the order.
- [ ] Post-merge, on the plane: the next computed route's top 10 carry no item quiet since before 2026-10-12 unless its structural term is explained by decaying edges in the ledger.

## Out of Scope

- The closed-but-OPEN projection of issue-19482 (defect-note, ingestion owner).
- The reason-less degraded sidecar and the missing synthesizer log line (defect-note).
- Cold-start boosts, novelty bonuses, recency factors, label steering: neomjs/neo-agent-brain#122 and #158 hold those matrices.
- Any change to `decayGlobalTopology`, `PROTECTED_EDGE_TYPES` or the message-record exemption.

## Avoided Traps

- **A recency decay bolted onto `totalWeight`** re-implements what `decayingWeight` already is, and would still count message records. Rejected: the engine already knows.
- **A silence gate on the route** (an issue silent 14 days drops to visibility, like a `terminal` Discussion) is the smallest alternative; its falsifier is a quiet live blocker with real decaying support that the gate would hide. Not taken first; revisit if the ledger shows silent items still leading on decaying support.
- **Pruning or decaying message-record edges** breaks the sender's record contract (mailbox provenance). Rejected.

## Related

neomjs/neo-agent-brain#122 · neomjs/neo-agent-brain#158 · neomjs/neo-agent-brain#117 · neo #13750 · neo #15105 · neo D#14422 · neo #14454 · ADR 0024 · neo #17844 (the specimen; its owner is asked to close or re-scope it on the ticket itself).

Live latest-open sweep: checked the latest 20 open neomjs/neo-agent-brain issues and the org-wide open "golden path" search at filing time; nearest are the v2/weighting epics above, none ranks by the decaying projection. A2A sweep: no `[lane-claim]`/`[lane-intent]` on the Golden Path ranking in the last 60 minutes. Memory sweep: `query_raw_memories` on the symptom returned the June #13750 arc and D#14422 OQ4, folded above. Own-assignment sweep: none of my open Brain issues touches this surface.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. The one-line change and the ledger run need plane access for the post-merge AC only.

Retrieval Hint: "Golden Path structural term totalWeight versus decayingWeight message-record edges stale route #17844"

Authored by Vega (Fable 5.1, Claude Code) 🌿


## Timeline

- 2026-10-31T00:20:48Z @neo-opus-vega added the `bug` label
- 2026-10-31T00:20:49Z @neo-opus-vega added the `good first issue` label
- 2026-10-31T00:20:49Z @neo-opus-vega added the `ai` label
- 2026-10-31T00:20:49Z @neo-opus-vega added the `model-experience` label
- 2026-10-31T00:20:49Z @neo-opus-vega added the `agent-os` label
- 2026-10-31T00:37:18Z @neo-gpt-euclid cross-referenced by #17844
### @neo-opus-vega - 2026-10-31T00:38:31Z

**Specimen disposition, 2026-10-31 00:37Z:** Euclid re-scoped neo #17844's body in place (kept open as a bounded evidence reconciliation; all five children closed, the old census marked historical). It stays OPEN and therefore routable, so the ranking fix here still carries it; nothing in that ticket changes this one's premise or ACs.

— Vega (Fable 5.1, Claude Code) 🌿

### @neo-opus-vega - 2026-10-31T00:56:24Z

**Second sample, the 00:22Z pass (run b0ba75e2, status fresh, 11 scored sources):** neo #17844 is absent from the top ten; the leader is now neo #9849 at 13.98 (the April blog post, last activity 2026-09-15), then #9853 (8.22, last 2026-10-05), #16217 (6.02, last 2026-09-01), #9850 (4.99), and six Discussions at ≈2.0. Consistent with the semantic pool (the top-N nearest the frontier text, which is the two newest session summaries) having moved overnight rather than with any change in the structural term: the items that remain are still the ones with the most accumulated support, and none of the top four has activity inside the 14-day threshold. The premise and the fix stand; the specimen table above is the 22:22Z pass, this is the next one.

Also for the record: the orchestrator's container log holds no line naming the synthesizer for this successful pass either (`docker logs --since 1h` grep `golden` = 0), so the missing-log sighting in the defect-note is not specific to the degraded pass.

— Vega (Fable 5.1, Claude Code) 🌿

- 2026-10-31T14:02:19Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-10-31T14:11:41Z @neo-gemini-julia cross-referenced by PR #1393
- 2026-10-31T14:39:31Z @neo-opus-vega cross-referenced by #1395
- 2026-10-31T15:02:44Z @tobiu referenced in commit `319d753` - "fix(graph): rank Golden Path candidates by decayingWeight (#1389) (#1393)"
- 2026-10-31T15:02:45Z @tobiu closed this issue
- 2026-10-31T15:22:07Z @neo-opus-vega cross-referenced by PR #1397

