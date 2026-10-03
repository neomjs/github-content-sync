---
id: 478
title: 'The cockpit''s state census: every surface × cold · live · stale · degraded · unreachable, as shipped'
state: CLOSED
labels:
  - documentation
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-vega
createdAt: '2026-10-03T08:23:30Z'
updatedAt: '2026-10-03T09:15:10Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/478'
author: neo-fable-clio
commentsCount: 0
parentIssue: 477
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[ ] 479 Row 2''s installed walkthrough: the five states provoked on one candidate, one receipt per state'
closedAt: '2026-10-03T09:15:10Z'
milestone: FM v1
---
# The cockpit's state census: every surface × cold · live · stale · degraded · unreachable, as shipped

The first leaf of #477 (FM v1 ROADMAP row 2): a read of the shipped cockpit that records, for every surface the operator can be looking at, what it renders in each of the five states — the state word, the reason, the next step — with the owning file and the symbol or literal that renders each sentence, stamped with the `dev` SHA the census read (line numbers decay before the walkthrough slot; `learn/` carries none today). The matrix is the artifact the walkthrough reads against and the only honest source of the row's remaining gap leaves.

## Context

Row 2's anchors are closed and its check has never run as one journey (#477's problem scope). Before provoking the five states on an installed candidate, the team needs to know what the product CLAIMS to show for each — otherwise the walkthrough measures surprise, not truth. The vocabulary exists in pieces: the deployment-state projection (`ok · stale · unavailable`), the banner's reason-carrying states (`SpineBanner.mjs`), the roster's and Home's `No agents yet`, the connect card's refusal words (#425 / #447), the activity feed's degraded composite (#263). Nobody has laid them side by side.

## The Problem

Without the census, a gap is an opinion. With it, a gap is a cell: a surface that renders a state word without its reason, a next step that names no action, a `stale` that is indistinguishable from `live`, a state a surface cannot enter at all. Each such cell is one leaf; the census is how they get filed on evidence.

## The Architectural Reality

- Surfaces: the spine banner, the roster card, Home's live field, the activity feed, the mailbox pane, the memories pane, the tasks pane, the Observatory, the Accounts view, the setup card (`apps/agentos/view/**`).
- States and their producers: the brain-health wire and the deployment-state projection (`ok · stale · unavailable`), the plane-attach probe's refusal classes, the fleet wake stream's `disconnected` lane, the feed sources' per-source failure (#263), the instance switch's binding (#181).
- The truth model from #15: answered causes retained, withdrawn on both loss transitions (thrown and absent); wired surfaces keep stale/live semantics.

## The Fix

One document under `learn/` (beside `TheNativeVessel.md` and `CockpitTour.md`): a matrix — rows = surfaces, columns = the five states (+ "one feed source failing" as the sixth column, #263's case) — each cell holding the rendered words (state · reason · next step) or `cannot enter` or `shows nothing`, with the owning file and the symbol or literal (a method, a config key, the string itself) that renders the sentence; the document head carries the `dev` SHA the census was read at. A short head names the vocabulary sources above and the rule a cell is judged by (a state word alone is not truth; reason and next step make it one). No code change. Cells that read as gaps are listed at the end as candidates — NOT filed by this leaf; the walkthrough (#477's second leaf) confirms them on the installed candidate first.

## Contract Ledger

| Surface | Authority | Behavior | Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| the census document | #477 (row 2) | every surface × every state, each cell sourced to its owning file + symbol/literal, SHA-stamped | a surface that cannot enter a state says so; an unknown cell is marked unknown, never guessed | the document itself | the matrix's cells point at the shipped code; a reviewer re-reads a sample of cells against `dev` |

Decision Record impact: none (a read of shipped behavior; the truth model is #15's / ADR 0041's).

## Acceptance Criteria

- AC-1: the document exists under `learn/` with the matrix (every surface listed in The Architectural Reality × the six columns), each filled cell anchored by owning file + symbol or literal, the head stamped with the `dev` SHA the census read.
- AC-2: cells that cannot be determined from source are marked `unknown` with the reason (e.g. a state only a live plane produces), never filled from memory.
- AC-3: the candidate-gap list at the end names each gap as a cell reference and the word that is missing (state / reason / next step), and states that filing waits for the walkthrough's confirmation.

## Out of Scope

Fixing any gap (each is its own leaf, filed after the walkthrough). The walkthrough itself (#477's second leaf). The Observatory's own Q1–Q5 checks (#312).

## Avoided Traps

Writing the matrix from what the product SHOULD show. Filling a cell from a screenshot older than the candidate. Reading the census from the installed app instead of `dev` source — the census is what the code claims, the walkthrough (#479) is what the installed candidate shows; mixing the two hides the gap the pair exists to find. Anchoring by line number. Turning the census into a redesign of the vocabulary.

## Related

#477 (parent) · #15 · #263 · #181 · #237 · neomjs/neo#16824

Live latest-open sweep: the latest 20 open Institution issues read at 2026-10-03T08:22Z (#477 … #7), no census or matrix leaf exists. A2A: no claim on this surface in the last hour. Own-assignment: #351, #477. Structure map: N/A — a `learn/` document.

Claimed by Vega 2026-10-03 08:56Z (her epic-review Greenlit #477); the symbol-anchor wording above is her ask, folded by the steward.

Origin Session ID: fb9561d9-a0dd-4f35-912c-095864afbae4
Retrieval Hint: "cockpit state census matrix surfaces five states reason next step file line"


## Timeline

- 2026-10-03T08:23:32Z @neo-fable-clio added the `documentation` label
- 2026-10-03T08:23:32Z @neo-fable-clio added the `enhancement` label
- 2026-10-03T08:23:32Z @neo-fable-clio added the `agent-os` label
- 2026-10-03T08:23:32Z @neo-fable-clio added the `ai` label
- 2026-10-03T08:24:02Z @neo-fable-clio cross-referenced by #479
- 2026-10-03T08:24:11Z @neo-fable-clio added parent issue #477
- 2026-10-03T08:24:16Z @neo-fable-clio added this to the **FM v1** milestone
- 2026-10-03T08:24:43Z @neo-fable-clio cross-referenced by #480
- 2026-10-03T08:28:43Z @neo-fable-clio cross-referenced by PR #482
- 2026-10-03T08:55:55Z @neo-opus-vega cross-referenced by #477
- 2026-10-03T08:56:04Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-10-03T08:59:28Z @neo-fable-clio marked this issue as blocking #479
- 2026-10-03T09:03:02Z @neo-opus-vega cross-referenced by PR #489
- 2026-10-03T09:15:10Z @tobiu referenced in commit `59ae695` - "docs(learn): the cockpit's state census — every surface × six states as shipped, anchored by file and symbol (#478) (#489)"
- 2026-10-03T09:15:10Z @tobiu closed this issue
- 2026-10-03T09:16:18Z @neo-opus-vega cross-referenced by #491
- 2026-10-03T10:56:50Z @neo-opus-grace cross-referenced by #498

