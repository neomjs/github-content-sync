---
id: 60
title: 'Agent detail rail IA: identity row, one state ledger, capacity gating, tab-seam pop-out'
state: CLOSED
labels: []
assignees: []
createdAt: '2026-08-29T23:34:22Z'
updatedAt: '2026-08-30T00:00:49Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/60'
author: neo-fable-clio
commentsCount: 0
parentIssue: 23
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-30T00:00:49Z'
---
# Agent detail rail IA: identity row, one state ledger, capacity gating, tab-seam pop-out

## Problem Scope

Implementation cut B of the #23 IA sketch: the agent-detail rail still rendered the pre-sketch shape — the name clipping to "E…" beside a fully spelled model tag (inverted importance), six state lines wrapping mid-phrase across three vocabularies (italic telltale prose, coloured source lines, a dangling producer suffix), a "Pop out de…" button floating OVER the identity content, a permanently-"not reported" throttle line whose adapter documents that no truth source exists, and per-section "not observed / awaiting live feed" boilerplate told twice.

## Solution Shape

Consume the sketch's detail sections verbatim: identity as ONE grid row (64px grid-locked avatar — size encodes drill depth: the card tier wears 40px, and the operator ruled 2026-08-30 that a detail portrait at-or-below card size inverts the hierarchy; the NAME is the only display-tier and only ellipsizing node; model chip + handle subordinate), ONE state ledger in the pane's own freshness-pill vocabulary (axis · pill rows; provenance/reasons on pill titles; tone classes reuse the freshness family), the throttle axis renamed `capacity` AND source-gated (renders only when a producer reports; model field/enum/adapter seam unchanged), the pop-out verb moved onto the tab header bar's action seam as a quiet icon (state names on title + aria), and the pane boilerplate collapsed into the head's freshness pill (feed-gated bodies stay empty; the awaiting truth rides the pill title).

## Acceptance Criteria

- [ ] Identity: no state line wraps at rail widths 240–360; the name is the only ellipsizing node; no control overlaps content (the pop-out lives on the tab header action seam, icon-only, title+aria named).
- [ ] Every liveness/wiring axis renders exactly once, as a ledger row in the freshness-pill vocabulary; producer literals and consumer reasons ride pill titles (inert attribute strings; the html-sink guard holds).
- [ ] `capacity` (ex-throttle) renders ONLY when observed; the unreported case renders no row.
- [ ] Feed-gated pane bodies are empty; the awaiting truth rides the freshness pill's title; the lane pane keeps its real body.
- [ ] Full unit + component batteries green; visual goldens re-verified; baseline stamp index-true.

## Out of Scope

The cockpit bar (cut A) · new data sources or verbs · the §04 chip-family unification (#24 leaf) · drawer-section CONTENT design (their feeds' own leaves).

## Related

Parent: #23 (ACs 4–5 of its checklist). Sketch: PR #57. Reviewer falsifier heritage: the #23 measurements (2026-08-22) + the operator's live captures (2026-08-30).

Authored by Clio (Fable 5, Claude Code). Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604

Retrieval Hint: `query_raw_memories("agent detail ledger freshness pills capacity source gated pop-out tab header")`



## Timeline

- 2026-08-29T23:35:45Z @neo-fable-clio cross-referenced by #23
- 2026-08-29T23:56:07Z @neo-fable-clio cross-referenced by PR #61
- 2026-08-30T00:00:49Z @tobiu referenced in commit `f97187a` - "Merge pull request #61 from neomjs/agent/60-detail-rail-ia

feat(agentos): agent-detail rail IA — identity row, state ledger, capacity gating, tab-seam pop-out (#60)"
- 2026-08-30T00:00:50Z @tobiu closed this issue
- 2026-09-01T20:57:22Z @neo-fable-clio cross-referenced by #66
- 2026-09-01T20:58:17Z @neo-fable-clio cross-referenced by PR #65
- 2026-09-01T21:42:14Z @neo-fable-clio cross-referenced by PR #70
- 2026-09-01T22:53:46Z @neo-fable-clio cross-referenced by #73
- 2026-09-01T23:30:31Z @neo-fable-clio cross-referenced by PR #75

