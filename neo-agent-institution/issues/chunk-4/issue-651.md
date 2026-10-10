---
id: 651
title: The Fleet Manager shows the handoff's Focus (v1) section
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-vega
createdAt: '2026-10-09T22:07:36Z'
updatedAt: '2026-10-09T22:07:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/651'
author: neo-opus-vega
commentsCount: 0
parentIssue: null
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 957 A focus section in the Sandman handoff: v1 rows, views, reach, the word'
blocking: []
---
# The Fleet Manager shows the handoff's Focus (v1) section

## Context

[D#19493](https://github.com/neomjs/neo/discussions/19493) OQ-8 graduated the overview as one focus slice with three readers:
- **the producer** (neomjs/neo-agent-brain#957): a `## Focus (v1)` section in the Sandman handoff, which the Fleet envelope carries as its own `focus` field;
- **the `/overview` skill** (neomjs/neo-agent-skills#152);
- **a pane in the Fleet Manager** — this ticket.

The closed cut line puts this pane **after v1**, outside #312's v1 close, so row 3's `passed` never waits on it (row 3's walker line, [D#19493 comment 18843205](https://github.com/neomjs/neo/discussions/19493#discussioncomment-18843205)). It is filed now so that OQ-8's targets exist.

## The Problem

Once neomjs/neo-agent-brain#957 lands, the answers to *where are we for v1, what is missing* and *which views still need love* exist in the handoff and on the Fleet wire. Nothing in the Fleet Manager shows them: a peer reads them through `get_sandman_handoff` or the skill, and the operator has no surface at all. The cockpit already renders the same handoff, but only one section of it.

## The Architectural Reality

- **The pane that reads the handoff today:**
  - `AgentOS.view.fleet.goldenpath.Container` (`apps/agentos/view/fleet/goldenpath/Container.mjs`) renders the `fleetGoldenPath` envelope;
  - it reads through `apps/agentos/util/GoldenPathEnvelope.mjs`, with the freshness facts row first and the producer's Markdown in a `MarkdownComponent`;
  - its source, neomjs/neo-agent-brain `ai/services/fleet/fleetGoldenPathSource.mjs:101–120`, extracts exactly `## Computed Golden Path (Strategic Recommendation)`.
- **Its second read:** neomjs/neo-agent-brain#957 adds the `## Focus (v1)` read and the envelope's `focus` field. It reports `handoff-section-not-found` when the producer has not written the section.
- **Stable keys:** views are keyed by dock item id and route, seats by Fleet agent id (#957 Fix §4). The views block's lines come from #505's sweep receipts.

## The Fix

1. **Where the section shows** is the design seat's call, in this ticket, before the PR (designed-surface rule). Recommended: a second section in the Golden Path pane, because it is the same handoff, the same freshness envelope and the same `MarkdownComponent`. The alternative is its own dock pane. The falsifier is the design read.
2. Render the `focus` field's Markdown with its per-block observation time and source link. A block that reads `unknown` shows its reason. Without the field (an older handoff, or `handoff-section-not-found`), the pane renders as it does today.
3. The words stay the producer's: a row's `Row state:` is shown raw (`ready` is not `passed`), and the pane derives no state of its own.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| The Fleet envelope's `focus` field (consumed) | neomjs/neo-agent-brain#957 (the producer, the Fleet source) | read as delivered; never recomputed | absent field → nothing new renders | #957's wire docs | unit arm with and without the field |
| The Focus section in the cockpit | this ticket + the design read | the four blocks with observation time and source link | `unknown` block → its reason | the pane's JSDoc | unit + one component arm; installed receipt |

Decision Record impact: none. Reporting only, aligned with the additive boundary neomjs/neo-agent-brain#957 keeps (ADR 0033).

## Acceptance Criteria

- [ ] AC-1 The design seat's read of where the section shows and of its words, in this ticket before the PR.
- [ ] AC-2 With a fixture envelope carrying `focus`, the cockpit shows the four blocks, each with its observation time and source link; an `unknown` block shows its reason (unit arm + one component arm).
- [ ] AC-3 Without the `focus` field, or with `handoff-section-not-found`, the pane renders as today with no error and no empty frame (unit arm).
- [ ] AC-4 A fixture row reading `ready` renders `ready`, never `passed` (unit arm).
- [ ] AC-5 *(installed, post-merge)* On the first candidate carrying neomjs/neo-agent-brain#957 and this leaf, the operator's two questions are answered from the cockpit in one read. Receipt on this ticket.

## Out of Scope

- The producer, the host-edge reader and the envelope field (neomjs/neo-agent-brain#957).
- The `/overview` skill (neomjs/neo-agent-skills#152).
- Any ranking change to the Golden Path (neomjs/neo-agent-brain#122).
- The view receipts themselves (#505).

## Avoided Traps

- **A second status authority.** The pane renders the producer's words and never promotes a row.
- **Waiting inside the v1 close.** This leaf is related to #312, not a native sub-issue of it, so the Observatory row's `passed` never waits on an after-v1 surface (the cut line's own placement).

## Related

D#19493 OQ-8 and OQ-1 (the cut line); neomjs/neo-agent-brain#957 (blocks this); neomjs/neo-agent-brain#123 (the parent of the focus slice); neomjs/neo-agent-skills#152; #312 (the Observatory epic; outside its v1 close); #505 (the view receipts).

Live latest-open sweep: checked the latest 20 open issues (created-descending) at 2026-10-09T22:06:40Z, plus a `Focus` keyword search; no equivalent. A2A in-flight claim sweep: the last 30 messages hold no claim on this pane; Clio's OQ-1 closure (21:27Z) names it as Vega's target. Memory Core rationale sweep: the only decision record is D#19493 OQ-8 and its STEP_BACK. Own-assignment sweep: #649 and #485, neither on this surface. Structure map (§1c): N/A, an Institution view with no `ai/` placement.

Not started: it waits for neomjs/neo-agent-brain#957 and for v1. Assigned to Vega per the OQ-1 target list.

Origin Session ID: 6764723e-7d1e-437a-81f7-cdc5be71402b

Retrieval Hint: "Focus (v1) section Fleet Manager pane renders handoff focus field after v1 Golden Path pane"

## Timeline

- 2026-10-09T22:07:36Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-09T22:07:38Z @neo-opus-vega added the `enhancement` label
- 2026-10-09T22:07:38Z @neo-opus-vega added the `agent-os` label
- 2026-10-09T22:07:38Z @neo-opus-vega added the `ai` label
- 2026-10-09T22:07:39Z @neo-opus-vega added the `design` label
- 2026-10-09T22:07:55Z @neo-opus-vega marked this issue as being blocked by #957

