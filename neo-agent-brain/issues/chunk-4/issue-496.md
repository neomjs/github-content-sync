---
id: 496
title: The Fleet Manager reads the computed Golden Path through one fleet-wire method
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-gpt-emmy
createdAt: '2026-09-25T15:48:45Z'
updatedAt: '2026-09-27T15:28:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/496'
author: neo-fable
commentsCount: 2
parentIssue: 122
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-09-27T15:28:17Z'
---
# The Fleet Manager reads the computed Golden Path through one fleet-wire method

## Context
Reopened for the operator's 2026-09-27 complete Golden Path requirement. [D#19151's accepted H1](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18615999) requires the human section whole and separately from `computed-route.v1`. Emmy owns this bounded follow-up; Institution #210 is the consumer.

## Problem and solution
The shipped `fleetGoldenPath` read carries only route, admission and REM. Reuse `get_sandman_handoff` through the existing admitted operation boundary and include its exact Computed Golden Path section in the same Fleet response. Extract from the canonical level-two heading through the next level-two heading; do not expose unrelated handoff sections or the server's filesystem path.

## Contract Ledger
| Surface | Behavior | Fallback / evidence |
|---|---|---|
| `fleetGoldenPath.handoff` | `{markdown, mtimeMs, ageMs, staleAfterMs, stale, reason}`; values are nullable, Markdown is exact source text | Explicit missing-section, unavailable-read or unwired reason; source tests |
| `sources.handoff` | Diagnostic `{state: 'available' · 'stale' · 'degraded' · 'unavailable', reason}` for the independent human read | Missing section is degraded; absent/unwired/failed read is unavailable; stale source remains readable. It does not change route capability. The consumer can also read `handoff.stale` and `handoff.reason` directly. |
| Route/admission/REM | Existing producer-owned values remain unchanged and independent | Handoff failure or staleness cannot replace their verdict |
| Plane attachment | Invoke existing `get_sandman_handoff` through `callHistoryOperation` / admitted plane client | No host filesystem fallback; wiring test |
| Source metadata | Preserve handoff mtime/age/staleness; mtime is not section capture time | Producer's capture line stays verbatim in Markdown |

## Acceptance Criteria
- [ ] Existing Fleet read returns the complete human section, including breakdowns/guard/interpretation when the producer writes them.
- [ ] Other handoff sections and filesystem paths are excluded.
- [ ] Missing, unreadable and stale human content has an explicit independent state while route/admission/REM retain their original values.
- [ ] Both Fleet boot paths use the existing operation boundary; focused source/wiring tests pass.


## Post-Merge Validation
The paired consumer and installed-app presentation remain owned by neomjs/neo-agent-institution#210. This reader's four implementation criteria above do not claim the installed UI is complete.

## Boundaries
No new MCP or Fleet method, scorer, ranking change, layout producer, local file fallback in plane mode, or synthesis. Preserve the existing typed route validator and admission. This completes the existing reader rather than reopening the Golden Path producer design.

Origin Session ID: f4539f98-814e-43c1-8214-a10206fb0d73

## Timeline

- 2026-09-25T15:48:45Z @neo-fable assigned to @neo-fable
- 2026-09-25T15:48:46Z @neo-fable added the `enhancement` label
- 2026-09-25T15:48:47Z @neo-fable added the `ai` label
- 2026-09-25T15:48:47Z @neo-fable added the `agent-os` label
- 2026-09-25T15:49:20Z @neo-fable added parent issue #122
- 2026-09-25T15:58:34Z @neo-fable cross-referenced by PR #499
- 2026-09-25T16:43:53Z @neo-opus-vega cross-referenced by #500
- 2026-09-25T17:01:29Z @neo-preview cross-referenced by PR #502
- 2026-09-25T17:06:04Z @neo-fable referenced in commit `2d7e916` - "fix(fleet): the golden path source reads its projection leaves at the use site (#496)

The wiring passed AiConfig.orchestrator.corpusProjection through as a config-shaped object (ADR 0019 B5, flagged by lint-config-template-ssot). The source now reads enabled / receiptPath / sourceRepository / sourceRef inline where the admission is evaluated; the factory takes an injectable readAdmission seam instead of a config subtree, and the spec drives it through that seam. Lint OK locally; fleet specs unchanged (111/112, the remRunStateDir boot-gate fixture stays the pre-existing red)."
- 2026-09-25T17:41:16Z @neo-preview cross-referenced by #503
- 2026-09-25T18:56:22Z @neo-opus-vega cross-referenced by PR #215
- 2026-09-25T19:03:10Z @neo-fable referenced in commit `e4059d2` - "fix(fleet): the golden path route crosses the plane boundary through get_computed_route (#496)

RA-2: the packaged shell runs the fleet server on the host in plane-attach mode, and the synthesizer's computed-route.json lives on the plane's handoff volume, so a host file read answered route-sidecar-missing by construction. A new Memory Core operation, get_computed_route (the get_sandman_handoff sibling: readComputedRoute is the file-contract half, the tool binds the resolved handoff path and reads the projection leaves inline), serves the validated sidecar with the computed-golden-path admission from the process that mounts the volume; the fleet source reads it through the same operation boundary as the REM state, so the envelope's three axes are one plane's. RA-1: the source spec, the dispatch ledger spec and the store spec join the unit job's run list. Ledger completion found by the run: the compliance tier map lacked the three community operations the OpenAPI declares (fails identically on dev)."
- 2026-09-25T19:20:54Z @tobiu referenced in commit `3655139` - "Merge pull request #499 from neomjs/feat/496-fleet-golden-path-read

feat(fleet): the cockpit reads the computed Golden Path through one wire method (#496)"
- 2026-09-25T19:20:54Z @tobiu closed this issue
- 2026-09-25T19:23:26Z @neo-fable cross-referenced by #122
- 2026-09-26T07:21:12Z @neo-fable-clio cross-referenced by #228
- 2026-09-26T07:26:59Z @neo-fable-clio cross-referenced by #533
### @neo-gpt-emmy - 2026-09-27T14:27:11Z

Reopening the existing Fleet Golden Path read for the missing human recommendation axis, under the operator's current correction and [D#19151's requirement to return the whole human section separately](https://github.com/neomjs/neo/discussions/19151#discussioncomment-18615999). Emmy takes this bounded completion; the original typed-route ranking/producer authority remains unchanged.

Reuse `get_sandman_handoff` through the already-admitted plane operation boundary. Return the exact Computed Golden Path Markdown section plus its source freshness beside the existing route/REM fields. No re-synthesis, no host filesystem fallback in plane mode, and no inference that handoff freshness establishes typed-route admission. Institution #210 is the paired consumer. Mnemosyne is currently rate-limited per the operator; this is completion of the shipped read, not a new Golden Path producer design.

- 2026-09-27T14:27:13Z @neo-gpt-emmy reopened this issue
- 2026-09-27T14:27:15Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-09-27T14:27:15Z @neo-gpt-emmy unassigned from @neo-fable
### @neo-gpt-emmy - 2026-09-27T14:27:16Z

**`[lane-override]` reassignment audit-trail** (#11537 §AC8)

**Previous assignees:** `@neo-fable`
**New assignees:** `neo-gpt-emmy`
**Reason:** Bounded completion of previously closed Fleet read under operator's current full-human-recommendation ruling and D19151; original assignee Mnemosyne unavailable until Friday per operator. No ranking/producer ownership transfer.

*Audit-trail per AGENTS.md §6.5 — `acknowledgedReassign` reason persistence. Graph-ingested via Retrospective daemon comment-scan path.*

- 2026-09-27T15:03:01Z @neo-gpt-emmy cross-referenced by PR #586
- 2026-09-27T15:28:17Z @tobiu referenced in commit `d5cd907` - "Merge pull request #586 from neomjs/codex/496-complete-gp-content

feat(fleet): expose complete Golden Path recommendation (#496)"
- 2026-09-27T15:28:17Z @tobiu closed this issue

