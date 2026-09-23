---
id: 413
title: The activity feed reads every corpus origin and qualifies rows by repository
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-fable-clio
createdAt: '2026-09-22T23:10:04Z'
updatedAt: '2026-09-23T01:17:31Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/413'
author: neo-fable-clio
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
closedAt: '2026-09-23T01:17:31Z'
---
# The activity feed reads every corpus origin and qualifies rows by repository

## Context

#246 / PR #410 (merged 2026-09-22, dev@2812f94) gave the fleet server's activity feed one declared root, `fleet.contentRoot` / `NEO_FLEET_CONTENT_ROOT`, read as `<root>/issues` + `<root>/pulls`. That is the pre-split single-origin layout — the `neo/` subtree of a `neomjs/github-content-sync` checkout, or the orchestrator's materialized root, which is single-origin by contract (`CORPUS_PROJECTION_ORIGIN = 'neo'`; `projectCoreCorpusIndex` keeps only that origin and strips the prefix — @neo-opus-vega, 2026-09-22). The corpus itself is five origins under `<repoSlug>/{issues,pulls,discussions,archive}` with one root `_index.json` whose rows carry `repoSlug` and a corpus-relative `path` (github-content-sync README; neomjs/neo#17846 §8.6). The Fleet Manager's own tickets live in `neo-agent-institution`, its Brain leaves in `neo-agent-brain`; a fleet whose activity feed shows only `neo` rows shows the wrong fleet.

## The Problem

- `wireFleetActivityReadSource.mjs` (`makeReadPrLaneSnapshot`) reads ONE `issuesDir` and ONE `pullsDir`; `readWorkGraphIssueRecords(issuesDir)` and `readSyncedPullRecords(pullsDir, {limit})` in `ai/services/graph/issueFocusSections.mjs` walk one tree each; `buildWorkGraphStallFindings({issuesDir, prs})` keys findings by a bare `issueId`.
- Across origins the bare number collides (`neo#86` vs `neo-agent-brain#86`), so a multi-origin read without qualified identity would merge unrelated conversations — the orphaning class #402's intake corrected on the KB side.
- The cockpit row renders `#17791` with no repository; with more than one origin on screen that is ambiguous. neomjs/neo#17846 §8.6a already names this feed as *the* consumer that decides the provenance grade (display, not filter) and records "the FM activity feed's direct read should become a declared leaf" (§8.7a step 3) — the leaf exists since #410; the origin walk is its second half.

## The Architectural Reality

- Owning directory: `ai/services/fleet/` (structure map run tonight: the read-source wiring, the PR/lane adapter, the composer live there); the record readers live in `ai/services/graph/issueFocusSections.mjs` and are shared with the Golden Path stall inference, so their signature is a consumed surface. No new `.mjs` file is required; if one is, its sibling is `fleetPrLaneActivityAdapter.mjs`.
- Layout detection is the corpus's own contract, not a guess: a root with `_index.json` whose rows carry `repoSlug` is multi-origin (`<root>/<repoSlug>/…`); a root with `issues/` directly under it is the legacy single-origin tree. Both must keep working — the second is what every deployment has today.
- The value of `NEO_FLEET_CONTENT_ROOT` therefore widens from "one origin's tree" to "the corpus root OR one origin's tree"; the leaf itself does not change (ADR-0019: no second leaf for a fact the tree already states).
- Identity: every record and event gains `repoSlug` (the conversation's origin repository — the field D#17846 §8.6a distinguishes from the KB's ownership tuple) and an origin-qualified id (`neo#19049`, `neo-agent-brain#402`); the default origin for the legacy layout is `neo` (the same constant the projection uses).

## The Fix

1. `wireFleetActivityReadSource.mjs`: resolve the origins under the root (index-declared when `_index.json` exists, else the single legacy origin), read issues + pulls per origin through the existing readers, stamp `repoSlug`, concatenate, hand the pure builder one combined set; the `limit` bound stays global (newest-first across origins), so a busy `neo` cannot starve a quiet `neo-agent-institution` row from being read at all — bound per origin before merge, then bound the merged set.
2. `issueFocusSections.mjs`: the readers accept an optional `origin` and qualify `issueId` / PR identity with it; unchanged callers (Golden Path stall inference on the single tree) keep today's bare ids because their origin is the default.
3. `fleetPrLaneActivityAdapter.mjs` / the snapshot DTO: events carry `repoSlug` and the qualified id under closed-set admission; an older cockpit that ignores the field sees exactly today's rows.
4. Honest degradation per origin: an origin whose tree is unreadable degrades naming the origin; the slot stays `wired` when at least one origin read and reports the failed origins in its capability reason (never a silent partial).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `NEO_FLEET_CONTENT_ROOT` value | `ai/configBase.mjs` `fleet.contentRoot` (unchanged leaf) | accepts a corpus root (`_index.json` + `<repoSlug>/…`) or one origin's tree | legacy tree = today's behavior, origin `neo` | leaf JSDoc + local Agent OS README | unit: both layouts against temp trees |
| activity events / PR-lane records | `fleetPrLaneActivityAdapter.mjs`, `wireFleetActivityReadSource.mjs` | `repoSlug` + qualified id on every event; per-origin read, global newest-first bound | absent field for legacy consumers | adapter JSDoc | unit: two origins sharing an issue number stay distinct (red-first) |
| `readWorkGraphIssueRecords` / `readSyncedPullRecords` | `ai/services/graph/issueFocusSections.mjs` | optional `origin` qualifies ids | callers without `origin` unchanged | function JSDoc | existing Golden Path arms green unchanged |
| cockpit activity row | `neo-agent-institution` (follow-up leaf) | renders `slug#N` for non-default origins, bare `#N` for `neo`, full `owner/repo#N` in the title | bare `#N` when the field is absent | Institution ticket | visual golden, filed there |

## Decision Record impact

none — aligned-with neomjs/neo#17846 §8.6a (display-grade provenance, this feed as the deciding consumer) and §8.7a step 3.

## Acceptance Criteria

- [ ] AC-1 With `NEO_FLEET_CONTENT_ROOT` pointed at a `github-content-sync` checkout ROOT, the PR/lane slot reads every origin present and every event and record carries `repoSlug` and an origin-qualified id.
- [ ] AC-2 The legacy single-origin layout (`<root>/issues`) behaves exactly as today, origin `neo`, ids unchanged for existing consumers (Golden Path arms green without edits).
- [ ] AC-3 Two origins sharing an issue number produce two distinct records and two distinct event ids (red-first arm). Stall findings are inferred for the Graph's origin only: the inference joins the Native Edge Graph by bare `issue-N`, and the Graph carries one origin by contract, so a foreign origin's number would join a stranger's node. *(Refined during implementation; the first wording asked for two stall findings.)*
- [ ] AC-4 An unreadable origin degrades the slot naming that origin while the rows of the other origins are kept; only a root with no readable origin takes the whole slot down. *(Refined: the slot reads `degraded`, not `wired` — the composer's `wired` means "we saw everything", and a partial read is not that; partial truth is carried under the honest word.)*
- [ ] AC-5 The bound applies after the merge, never per origin alone: a quiet origin's newest row is never cut by a busy origin's older rows (an arm with five old rows in one origin and one fresh row in another under `limit: 3`).
- [ ] AC-6 Host witness against the sparse corpus checkout used for #410 (all five origins fetched): the newest rows of at least two origins appear in one snapshot, cited by number and date in the PR.
- [ ] Post-merge: the Institution display leaf is filed with the DTO field names as shipped.

## Out of Scope

The cockpit rendering (Institution follow-up, filed after this DTO ships) · the plane's compose value for the env (`#213`) · the KB tenant (`#402` / `#411`) · the orchestrator projection's single-origin contract.

## Avoided Traps

- Reading the orchestrator's materialized root for multi-origin rows: it has one origin and no `<repoSlug>/` layer by contract (measured by @neo-opus-vega tonight); the corpus checkout or an installer tree is the multi-origin source.
- A second leaf for the origin list: the corpus root already declares its origins in `_index.json`; a csv leaf would drift from the tree it describes.
- Qualifying ids everywhere at once: the Golden Path stall inference reads the same readers on a single tree; the optional `origin` keeps its ids bare and its arms untouched.

## Related

Parent leaf: `#246` (closed by PR #410). Siblings: `#402` / `#411` (the KB's corpus tenant), `#213` (declarative deployment; the env's plane value), Grace's engine-side `resources/content` deletion under neomjs/neo#17416 (her installer's output directory is one valid value for the root). Design authority: neomjs/neo#17846 §8.6, §8.6a, §8.7a.

Live latest-open sweep: checked the latest 20 open issues of this repository at 2026-09-22T23:08Z; no equivalent (search "activity origin repoSlug": none). A2A in-flight sweep (last 30 at 22:29Z, last 10 at 22:45Z, all read-states): no claim on this leaf; @neo-opus-vega's 22:07Z sentence and my PR-open broadcast both name it as the next FM leaf. Memory sweep (`query_raw_memories` on the problem's nouns): no prior decision beyond D#17846. Own-assignment sweep: `#37`, `#50`, `#51`, `#53` — no overlap. Structure map: `npm run ai:structure-map -- --files --loc` run 2026-09-22; `ai/services/fleet/` is the owning folder, no new file planned.

Assignee: @neo-fable-clio (the FM lead's leaf; implementation in a fresh session).

Authored by Clio (Claude Fable 5.1, Claude Code).
Origin Session ID: cf6c8297-03b1-41af-8d76-cb19eb9aa9c4
Retrieval Hint: `query_raw_memories("fleet activity feed multi-origin repoSlug qualified id corpus root index")`


## Timeline

- 2026-09-22T23:10:05Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-09-22T23:10:05Z @neo-fable-clio added the `enhancement` label
- 2026-09-22T23:10:06Z @neo-fable-clio added the `ai` label
- 2026-09-22T23:10:06Z @neo-fable-clio added the `agent-os` label
- 2026-09-22T23:29:12Z @neo-opus-vega cross-referenced by #415
- 2026-09-22T23:32:45Z @neo-fable-clio cross-referenced by PR #416
- 2026-09-23T00:22:47Z @neo-fable-clio referenced in commit `4bee719` - "fix(fleet): the readers own origin identity and the PR bound ranks by event time (#413)

Origin identity moves to where the records are made: qualifyOriginId lives in the corpus projection contract, and both shared readers take an optional origin — an issue record answers an origin-qualified issueId and a pull record a prId, each with repoSlug, while callers without an origin keep the bare ids the Graph keys. The pull reader's pre-parse bound ranked candidates by the filename's PR number, which dropped the newest event whenever an older PR was updated after a newer one; it now ranks by updatedAt peeked from each file's head through one bounded descriptor read, still parsing only the bounded set. The wiring hands the origin to the readers instead of stamping records afterwards, and the adapter keys events by the contract's helper. The Brain Unit smoke list executes the touched fleet, graph and config specs instead of only collecting them."
- 2026-09-23T00:30:19Z @neo-fable-clio referenced in commit `9b4f523` - "feat(fleet): the activity feed reads every corpus origin and qualifies rows by repository (#413)

The PR/lane slot read one issues tree and one pulls tree under the fleet.contentRoot leaf, so a fleet whose conversations live in five repositories showed only the Graph's origin. The wiring now takes the root itself and resolves its origins from the tree and the corpus's own index: a root with issues/ directly under it stays the single-origin tree every deployment reads today (the engine tree, one origin's subtree, the orchestrator's materialized root), and a corpus checkout root reads <root>/<repoSlug>/{issues,pulls} for every slug the index declares, the Graph's origin first. Records carry their origin, and every event id of a foreign origin is qualified (<repoSlug>#<number>) while the Graph origin's ids stay byte-identical, so the same number in two repositories never merges. Origins are read inside their own containment: one unreadable origin degrades the slot by name while the rows of the others are kept, and only a root with no readable origin takes the slot down. Stall inference stays with the Graph's origin, because its joins are by bare issue id and the Graph carries one origin by contract. Both devFleetServer wiring sites hand the wiring the leaf's value instead of two derived directories."
- 2026-09-23T00:30:19Z @neo-fable-clio referenced in commit `f419ad4` - "fix(fleet): the readers own origin identity and the PR bound ranks by event time (#413)

Origin identity moves to where the records are made: qualifyOriginId lives in the corpus projection contract, and both shared readers take an optional origin — an issue record answers an origin-qualified issueId and a pull record a prId, each with repoSlug, while callers without an origin keep the bare ids the Graph keys. The pull reader's pre-parse bound ranked candidates by the filename's PR number, which dropped the newest event whenever an older PR was updated after a newer one; it now ranks by updatedAt peeked from each file's head through one bounded descriptor read, still parsing only the bounded set. The wiring hands the origin to the readers instead of stamping records afterwards, and the adapter keys events by the contract's helper. The Brain Unit smoke list executes the touched fleet, graph and config specs instead of only collecting them."
- 2026-09-23T01:17:31Z @tobiu referenced in commit `1fc890c` - "Merge pull request #416 from neomjs/agent/413-multi-origin-activity-feed

feat(fleet): the activity feed reads every corpus origin and qualifies rows by repository (#413)"
- 2026-09-23T01:17:32Z @tobiu closed this issue

