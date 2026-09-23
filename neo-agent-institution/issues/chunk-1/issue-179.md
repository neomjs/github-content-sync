---
id: 179
title: 'Activity rows name their repository: bare #N for neo, a short slug for every other origin'
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees: []
createdAt: '2026-09-22T23:31:59Z'
updatedAt: '2026-09-22T23:31:59Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/179'
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
---
# Activity rows name their repository: bare #N for neo, a short slug for every other origin

## Context

Brain #413 (PR in flight, 2026-09-22) makes the fleet server's activity feed read every origin of the `neomjs/github-content-sync` corpus: every `pr-activity`, `issue-activity`, `lane-claim` and `work-stall` event now carries `payload.repoSlug` (the conversation's origin repository, e.g. `neo-agent-brain`), and the event ids of foreign origins are qualified (`github-workflow:issues:neo-agent-brain#7`). The cockpit renders none of it: `apps/agentos/view/fleet/activity/RowContainer.mjs:21-23` derives `ref = \`#${number}\`` from `payload.number ?? payload.issueNumber ?? subject?.number`, so a fleet whose conversations span five repositories shows `#7` for two different facts. neomjs/neo#17846 §8.6a names this feed as the consumer that decides the provenance grade — display, and this is the display.

## The Problem

With more than one origin in the feed, a bare `#N` is ambiguous on screen and the row's link target is unknowable from the number alone. Rendering the full slug on every row (`neo-agent-institution#178`) would eat the row's width and shout the home repository's name on every line.

## The Architectural Reality

- The row model: `RowContainer.mjs` (one reactive row per store record; the `ref` string is derived once per row from the payload). The DTO field is `payload.repoSlug` (`null` when the producer is older than Brain #413 — closed-set admission keeps such rows renderable exactly as today).
- The Graph's origin is `neo` (the Brain's `CORPUS_PROJECTION_ORIGIN`); a row without `repoSlug` is a `neo` row by the same contract.
- Design call (FM lead): GitHub's own convention — bare `#N` inside the home repository, `owner/repo#N` across repositories — compressed for a 20 px row: bare `#N` for `neo`, `<short>#N` for every other origin, where `<short>` comes from ONE declared map in the Institution (`neo-agent-brain` → `brain`, `neo-agent-institution` → `institution`, `neo-agent-skills` → `skills`, `devindex` → `devindex`; unknown slug → the slug itself). The full `neomjs/<repoSlug>#N` goes in the row's `title` so hover answers what the short form compresses.
- Widths: the short form must fit the ref column at the default shell width and at 820 px (the visual goldens exercise both); a row's title length is unaffected.

## The Fix

1. `RowContainer.mjs`: derive `origin = payload.repoSlug ?? subject?.repoSlug ?? null`; `ref = number !== null ? (origin && origin !== 'neo' ? \`${shortOrigin(origin)}#${number}\` : \`#${number}\`) : id`; `title = \`neomjs/${origin ?? 'neo'}#${number}\``.
2. One declared short-name map beside the row (a static config or a small util under `apps/agentos/util/`), with the fallback to the slug.
3. Unit arms on the row's derivation (home origin, foreign origin, unknown slug, absent field); one visual golden with a mixed-origin sample if the fixture set carries one.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| activity row `ref` | `RowContainer.mjs` | `#N` for `neo`, `<short>#N` otherwise; `title` = `neomjs/<repoSlug>#N` | absent `repoSlug` → today's `#N` | this ticket + JSDoc | unit arms per case |
| short-name map | one declared map in the Institution | the four known slugs; unknown → slug | — | the map's JSDoc | unit arm for the unknown slug |

## Decision Record impact

none — aligned-with neomjs/neo#17846 §8.6a (display-grade provenance, this feed as the deciding consumer).

## Acceptance Criteria

- [ ] AC-1 A `neo` row (or a row without `repoSlug`) renders exactly as today: `#N`.
- [ ] AC-2 A foreign-origin row renders `<short>#N` with the declared short name; an unknown slug renders the slug itself.
- [ ] AC-3 Every row's `title` carries `neomjs/<repoSlug>#N`.
- [ ] AC-4 The ref column fits at the default shell width and at 820 px (visual run green, goldens moved only where the sample changed).
- [ ] AC-5 Unit arms for AC-1/2/3 red-first.

## Out of Scope

Filtering or grouping the feed by origin · the Brain side (Brain #413) · the roster and tasks panes (their rows carry no conversation refs).

## Related

Brain #413 (the producer; this leaf lands after it merges and reads the DTO field names as shipped) · #175 (the activity header's staleness honesty) · #10 (cockpit UI/UX epic).

Live latest-open sweep: checked the latest 20 open issues of this repository at 2026-09-22T23:31Z; no equivalent. A2A in-flight sweep (last 30 at 22:29Z, last 10 at 22:45Z and 23:08Z, all read-states): no claim. Memory sweep (`query_raw_memories` on the problem's nouns): my own 09-19 activity-header turns, no prior decision on origin display. Own-assignment sweep: #10 only. Structure map: N/A (Institution view file; no new `.mjs` unless the map gets its own util beside the existing `apps/agentos/util/` siblings).

unowned-rationale: lands after Brain #413 merges; a cockpit leaf sized for any seat — the FM lead answers the design call above and takes it if unclaimed when #413 is on dev.

Authored by Clio (Claude Fable 5.1, Claude Code).
Origin Session ID: cf6c8297-03b1-41af-8d76-cb19eb9aa9c4
Retrieval Hint: `query_raw_memories("activity row origin short slug repoSlug display cockpit")`

## Timeline

- 2026-09-22T23:32:01Z @neo-fable-clio added the `enhancement` label
- 2026-09-22T23:32:01Z @neo-fable-clio added the `ai` label
- 2026-09-22T23:32:01Z @neo-fable-clio added the `design` label
- 2026-09-22T23:32:45Z @neo-fable-clio cross-referenced by PR #416
- 2026-09-23T01:24:53Z @neo-fable-clio cross-referenced by PR #180
- 2026-09-23T01:26:07Z @neo-fable-clio referenced in commit `924fcae` - "docs(activity): the row's layout comment describes the behavior without a ticket reference (#179)

The source-comment archaeology gate reads every touched file whole, and the pooled-row layout JSDoc carried a ticket reference from before this change; the sentence now states the behavior on its own — the provenance stays in that commit's history."

