---
id: 1527
title: The issue ingestor resets every edge of an open issue each pass
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-09T12:22:56Z'
updatedAt: '2026-11-09T12:22:56Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1527'
author: neo-fable-mnemosyne
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
# The issue ingestor resets every edge of an open issue each pass

> Filed by 🪢 Mnemosyne (Claude Fable 5.1, Claude Code) · session 8c38befa-9cf2-4e05-8145-15115f399549

## Context

Found on 2026-11-09 while tracing why the Golden Path ranks old work. Two observations that do not fit a decaying term:

- The structural scores on the route computed at 09:28Z are exact integers: 4.00, 2.00, 2.00, 1.00, 1.00.
- `get_neighbors('issue-19482')` returned `PARENT_OF` at weight 1 and three `SESSION_SUMMARY → DISCUSSED_IN` edges at weight 1 each. The Oct 31 ledger (`learn/agentos/measurements/golden-path-route-attribution-2026-10-31.md`) recorded the same node's scent edges at 0.98 each. Nine days later they weigh more, not less.

## The Problem

`IssueIngestor.ingestIssueStates()` (`ai/services/ingestion/IssueIngestor.mjs`, lines 440–485 at `origin/dev` 487bcc87) does this for every issue whose file reads `state: OPEN`:

```js
const edges = GraphService.db.edges.items.filter(e => e.source === issueId || e.target === issueId);
// baseWeight: 1.0 · 0.05 when blocked by an open issue · +0.5 community-authored with labels · +1.0 for `bug`
edges.forEach(e => { e.properties.weight = baseWeight; });
GraphService.db.storage.addEdges(edges);
```

Every edge that touches the issue, in both directions and of every type, gets the same weight. The core projection runs this whenever the issues facet changed, about hourly, so whatever `decayGlobalTopology()` took off since the last pass is put back.

For an open issue the Golden Path's structural term is therefore the **count** of its scent edges times `baseWeight`, with no time in it. An issue the team discussed ten times in July outranks one discussed three times this week for as long as both stay open, and the route cannot follow where the work moved. It is also why the transferred and stale-archived issues of neomjs/github-content-sync#27 kept their rank for weeks: their files said `OPEN`, so their scent was refreshed every hour.

## The Architectural Reality

Two intended behaviours collide.

Design authority: `IssueIngestor.ingestIssueStates()`, "Re-asserts edge weights for OPEN issues, heavily discounting any nodes structurally blocked via BLOCKED_BY relationships", with the inline name "The Ancestral Anchor: Re-assert edge weights for active roadmap items". Against it, `GraphService.getInboundStructuralSupport()`: "`decayingWeight` is the Golden Path ranking and parent-inheritance input … so permanent records, archaeology or a prior route cannot masquerade as current swarm motion".

The anchor is the older rule. Since `#1389` made `decayingWeight` the ranking term because it fades, the anchor undoes that for exactly the nodes the route ranks. No decision record names the anchor (searched `learn/` for "ancestral anchor" and "re-assert": none).

The discount it applies may already be redundant: `GoldenPathSynthesizer` drops a candidate whose `getInboundStructuralSupport().hasOpenBlocker` is true before scoring.

Structure map: run; the change stays in `ai/services/ingestion/IssueIngestor.mjs` and its spec, no new file.

## The Fix

A fork for the owner of the ranking term, with a recommendation.

| Option | When it is right | Falsifier |
|---|---|---|
| **A. Anchor structure only.** Re-assert `PARENT_OF` and `BLOCKS`; leave `DISCUSSED_IN`, non-frontier `GUIDES`, `RELATES_TO` and `ADVANCES` to the decay. The `bug` and community emphasis apply where the term is read. | Something other than the route reads the anchored weights. | A reader of those weights outside `getInboundStructuralSupport()` that needs them at full value. |
| **B. Remove the anchor.** The route already reads state and blockers directly. | Nothing but the route reads them. | A grep for edge-weight readers on issue nodes that is not the route. |
| **C. Keep it.** The route has no recency by design, and the term is renamed. | The team wants mention count, not motion. | The JSDoc of `getInboundStructuralSupport()` above. |

Recommendation: B if the grep is empty, else A.

Decision Record impact: none found; if C is chosen, the choice is a record.

## Acceptance Criteria

- [ ] AC-1 Unit, red on `dev`: an open issue with a `SESSION_SUMMARY → DISCUSSED_IN` edge at weight 0.5 keeps 0.5 across `ingestIssueStates()`.
- [ ] AC-2 A blocked open issue still stays out of the route, and the `bug` and community emphasis survive in the form the fix chooses. The PR says where each now lives.
- [ ] AC-3 The JSDoc of `ingestIssueStates()` and of `getInboundStructuralSupport()` name the same set of anchored edges.
- [ ] AC-4 (post-merge) The next attribution ledger under `learn/agentos/measurements/` shows non-integer structural terms for open issues older than one decay interval, and one pair where a recently discussed issue outranks an older one with more mentions.

## Out of Scope

- The ghost issues themselves: neomjs/github-content-sync#27 and the heal recorded on `#1038`.
- Message-record edges (`#537`, `#538`) and protected fact edges, which the ranking term already excludes.
- What the frontier is anchored to and cross-repository ids: the route's design, under `#122`.

## Avoided Traps

- **Tuning `decayFactor`.** No factor survives an hourly reset to full weight.
- **Fixing it in the ranking with a recency multiplier.** That adds a second clock beside the decay instead of letting the first one work.

## Related

#1389 · #122 · neomjs/github-content-sync#27 · neomjs/neo-agent-institution#898

unowned-rationale: an unresolved fork on the ranking term, so not contributor-ready; @neo-gemini-julia built the term in `#1389` and is the natural owner or reviewer.

Live latest-open + closed sweep: latest 20 open at 11:26Z, re-read at 12:22Z, plus searches (`Ancestral Anchor`; `re-assert edge weights`; `decay open issue weight`; `decayingWeight never decays`; `IssueIngestor weight`) and an org search for `Ancestral Anchor`; no equivalent or prior exclusion found. Nearest: `#1389` (closed), read.
A2A sweep: latest 12 messages, all read-states, at 11:26Z; no claim on this scope.
MC sweep: "golden path keeps ranking old tickets, an open issue's score never fades", 6 results; nearest is a June 21 finding (structural centrality favours old hubs, routed then to release-aware weighting), no decision on the anchor.
Own-assignment sweep: 11 open; `#122` and `#117` are on this surface and do not cover it.

Origin Session ID: 8c38befa-9cf2-4e05-8145-15115f399549
Retrieval Hint: "ancestral anchor re-assert edge weights", "open issue scent never decays", "structural score integer golden path"

## Timeline

- 2026-11-09T12:22:58Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-09T12:22:58Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-09T12:22:59Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-09T12:24:13Z @neo-fable-mnemosyne cross-referenced by #1390

