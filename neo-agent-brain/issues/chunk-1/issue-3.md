---
id: 3
title: Calibrate community policy from measured evidence
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - performance
assignees:
  - neo-gemini-maryam
createdAt: '2026-07-14T05:32:05Z'
updatedAt: '2026-10-26T21:56:45Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/3'
author: neo-gpt-euclid
commentsCount: 1
parentIssue: 106
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy:
  - '[x] 101 Project bounded tenant community-attention counts'
  - '[x] 103 Expose a temporal community Bird View and seen state'
  - '[x] 15156 Push hosted GitHub community batches securely'
  - '[x] 104 Coordinate local GitHub community reconciliation'
blocking:
  - '[ ] 98 Prove the community-activity authority chain end to end'
  - '[ ] 100 Enable metric-gated community-steward wake leases'
---
# Calibrate community policy from measured evidence

## Context

OQ10 forbids intuition-derived production thresholds. After shadow and live paths exist, this leaf converts measured evidence into explicit policy or an explicit decision to leave a value unset.

This is one fully closeable PR leaf under Epic neomjs/neo-agent-brain#106. The live parent-child and blocked-by graph is authoritative; this body owns only this leaf's contract.

## The Problem

Cadence, page/concurrency bounds, retention/archive policy, TTLs, steward activation, wake thresholds, and optional topology choices have different cost and failure drivers. Choosing them incidentally in implementation PRs hides the decision and makes later drift unauditable.

## The Architectural Reality

This is an evidence/decision leaf, not an instrumentation rewrite. It consumes versioned shadow reports plus live admission, local/hosted connector, Bird View, count, claim, and health metrics. Any config changes must obey ADR 0019 and read resolved leaves at use sites. ADR 0036 is updated/amended with the measured disposition.

The Agent OS structure map was run on 2026-07-14. New service/script/test placement must use the named sibling-file-lift fast paths; no service logic moves into MCP server entrypoint directories.

## The Fix

Run the named observation window, publish a citeable calibration report, select or explicitly reject each threshold, add only evidence-backed config/policy surfaces, and record revalidation decisions for E/F/K/O/V/X. A rejected steward activation makes the wake child close not planned rather than inventing another delivery shape.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| Calibration report | OQ10 metric list | Versioned sources, denominators, coverage, confidence, and decision per value | Insufficient evidence leaves value unset | ADR/runbook | Reproducible report |
| Operational policy | ADR 0036 | Only evidence-backed cadence/page/storage/TTL/steward/wake settings | No hidden defaults; unsafe values remain disabled | Config docs | Boundary tests |
| Deferred options | Convergence table | E/F/K/O/V/X each revalidated against its named trigger | No trigger means disposition unchanged | ADR update | Option-by-option review |

## Decision Record impact

Depends on ADR 0036 and measured end-to-end substrates; may amend ADR 0036's operational values but cannot supersede its authority boundaries.

## Decision Record

**Required: ADR 0036.** This leaf is not code-ready until the ADR-0036 child of neomjs/neo-agent-brain#106 is accepted at the human merge gate.

## Discussion Criteria Mapping

| Upstream graduated criterion | This leaf's executable contract |
|---|---|
| OQ10 | Selects or leaves unset cadence, pagination, retention, TTL, and cost limits from measurements. |
| OQ7 J | Determines whether steward lease/wake is justified and at what bounded thresholds. |
| K/O | Activates only on non-reconstructable acquisition or measured hosted backpressure. |
| E/F/V/X | Keeps accelerators/self-service deferred absent their explicit authority/value triggers. |

Source authority: Discussion #15139 body at the version-bound graduation anchor plus Grace's [STEP_BACK](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631120) and [GRADUATION_APPROVED](https://github.com/neomjs/neo/discussions/15139#discussioncomment-17631315).

## Acceptance Criteria

- [ ] **AC1** — The report cites exact shadow/live source manifests, windows, denominators, coverage gaps, and repeatability.
- [ ] **AC2** — It measures pages/event, duplicate/revision/tombstone rate, admission conflicts/receipt hits, checkpoint bytes, storage growth, latency, hook bytes/wakes per event, steward vacancy, and time-to-ack/claim/response.
- [ ] **AC3** — Every cadence, page/concurrency, retention/archive, TTL, steward, and wake value is evidence-backed or explicitly unset.
- [ ] **AC4** — No hidden fallback/default is introduced in services or config.
- [ ] **AC5** — E/F/K/O/V/X receive explicit keep-deferred, activate, or reject dispositions tied to their graduated revalidation triggers.
- [ ] **AC6** — Any AiConfig surface follows ADR 0019 sanctioned patterns and has boundary tests.
- [ ] **AC7** — ADR 0036 and operator docs record the resulting policy and next revalidation trigger.
- [ ] **AC8** — If steward wake is not justified, the wake child is closed not planned with this evidence.

## Out of Scope

New acquisition families, connector implementation, a queue/outbox without triggered evidence, or implementation of the wake itself.

## Avoided Traps

Do not use the original 30-day lower bound as complete truth, select round numbers by intuition, hide defaults, or convert rare signal into all-resident interruption.

## Related

- Parent: neomjs/neo-agent-brain#106
- Source: Discussion #15139 OQ7/OQ10 and option gates
- Authority: ADR 0036, ADR 0019

Origin Session ID: 837ad74b-c2d2-413d-9aab-b7165a93a82a

## Handoff Retrieval Hints

- `community activity calibration cadence retention TTL steward wake`
- `Discussion 15139 OQ10 metrics revalidation K O V X`


## Creation Freshness

Creation duplicate sweep: immediately before filing at 2026-07-14T05:32:04.397Z, checked the latest 20 open issues and last 30 all-state A2A messages. The independent broader audit at 2026-07-14T05:13:00Z covered open and closed issues, pull requests, A2A, ADRs, and code; no equivalent owner or foreign claim existed.

## Timeline

- 2026-07-14T05:32:05Z @neo-gpt-euclid added the `documentation` label
- 2026-07-14T05:32:06Z @neo-gpt-euclid added the `enhancement` label
- 2026-07-14T05:32:06Z @neo-gpt-euclid added the `ai` label
- 2026-07-14T05:32:06Z @neo-gpt-euclid added the `architecture` label
- 2026-07-14T05:32:06Z @neo-gpt-euclid added the `performance` label
- 2026-07-14T05:33:05Z @neo-gpt-euclid marked this issue as being blocked by #15156
- 2026-07-14T05:34:32Z @neo-gpt-euclid cross-referenced by #106
- 2026-07-19T00:44:18Z @neo-gpt-emmy cross-referenced by #15526
- 2026-08-24T11:38:03Z @neo-gpt-emmy cross-referenced by #17701
- 2026-08-24T11:39:12Z @neo-gpt-euclid marked this issue as being blocked by #15156
- 2026-08-26T15:11:06Z @neo-gpt-euclid marked this issue as blocking #98
- 2026-08-26T15:11:43Z @neo-gpt-euclid marked this issue as blocking #100
- 2026-08-26T15:13:19Z @neo-gpt-euclid marked this issue as being blocked by #101
- 2026-08-26T15:13:31Z @neo-gpt-euclid marked this issue as being blocked by #104
- 2026-08-26T15:13:34Z @neo-gpt-euclid marked this issue as being blocked by #103
- 2026-08-26T15:14:31Z @tobiu added parent issue #106
- 2026-08-26T15:17:45Z @neo-opus-ada cross-referenced by #136
- 2026-09-02T02:41:33Z @neo-opus-grace cross-referenced by #298
- 2026-09-19T23:01:07Z @neo-gpt-euclid cross-referenced by PR #392
- 2026-10-05T13:58:56Z @tobiu referenced in commit `a38f232` - "fix(sync): the GitHub Workflow sync stops deriving Portal indexes and SEO (#888) (#889)

The derive step, `SyncService.rebuildContentIndexesAndSeo`, ran only in the manual modes
(`runFullSync` and `--emit-only`). The one scheduled caller, github-content-sync's
publish-corpus, runs `--corpus-only`, which never derived. The Portal's deploy now derives
from the published corpus (neomjs/pages#14). The step also called the engine with `{root}`
alone, so moving the pin past neomjs/neo#19408 would have made it throw.

- SyncService: `rebuildContentIndexesAndSeo` and the `deriveContent` option are removed, and
  `emitGeneratedContentAndDerive` becomes `emitGeneratedContent`. The auto-push allowlist
  drops the three `apps/portal/*` paths that only the derive wrote.
- syncGithubWorkflow.mjs: the docblock names the real scheduled caller and its mode.
- Specs: the derive arms are removed. Stage2 counts passes with `pullRuns`, and its
  allowlist arm asserts that no Portal path remains. RechunkOrdering keeps its property,
  re-anchored on the aggregate verdict. Its provenance paragraph moves here: the spec is
  half B of arm #3 of the Engine's RebuildContentIndexesAndSeo.spec.mjs (deleted by
  neomjs/neo@c623b2f63c), whose half A stays Engine-side under neomjs/neo#17922 (#298)."
- 2026-10-15T11:28:33Z @neo-opus-ada cross-referenced by #1049
- 2026-10-26T21:56:42Z @neo-gemini-maryam assigned to @neo-gemini-maryam
### @neo-gemini-maryam - 2026-10-26T21:56:45Z

### Ticket Intake & Sharpening (`#3` — Calibrate community policy from measured evidence)

- **Epic-Review Pre-Requisite:** Satisfied by `@neo-opus-ada`'s review (`#15145#issuecomment-5036518069`) and non-author Gemini-family `/epic-review` + Gemini Liveness Revalidation on `neomjs/neo-agent-brain#106#issuecomment-6426058715` [QUARANTINED_URL: github.com] (`commentId: IC_kwDOUBzDFM8AAAABfwXf2w`, verdict: `Greenlight`).
- **Ticket Reality Classification:**
  - `Ticket age`: `createdAt: 2026-07-14T05:32:05Z`, `updatedAt: 2026-08-24T11:39:03Z`
  - `Bot stale-band`: `in-stale-window` (`stale: false`, `no auto close: false`)
  - `Currency / successor-risk evidence`: Checked merged leaves (`#15473` ADR 0036, `#15479` shadow measurement `#15149`, `#381` registration `#15150`, `#383` admission `#15151`, `#385`/`#387`/`#388` GitHub family reconcilers `#15152`, `#393` hosted connector `#105`, `#395` local coordinator `#104`, `#396` Bird View/seen `#103`) and stacked open PRs (`#1307` canonical Task claim `#102`, `#1308` bounded attention counts `#101`). All upstream measurement and runtime seams required by `#3` are now in place.
  - `ADR successor-risk`: `adr-amendment-required — artifact neomjs/neo-agent-brain#3 (2026-07-14); ADR 0036 (Accepted 2026-07-18) & ADR 0019 (Accepted 2026-06-04); evidence learn/agentos/decisions/0036-durable-community-activity-authority.md + learn/agentos/measurements/community-activity-shadow-2026-07-18.md; route amendment-required (amends ADR 0036 with §9 Operational Calibration & Option Revalidation while preserving all ADR 0036 authority boundaries and ADR 0019 read-at-use-site / zero-hidden-default invariants).`
  - `Verdict`: `valid-as-written` (accepted and sharpened below).
- **Prescription & Structural Pre-Flight:**
  - `Prescription checked`:
    - `ai/scripts/maintenance/communityActivityPolicyCalibrationCore.mjs` + `ai/scripts/maintenance/calibrateCommunityActivityPolicy.mjs` — matches sibling pattern of `ai/scripts/maintenance/communityActivityShadowProbeCore.mjs` + `ai/scripts/maintenance/probeCommunityActivityShadow.mjs` (`sibling-file-lift`).
    - `ai/daemons/orchestrator/scheduling/communityReconciliation.mjs` + `ai/daemons/orchestrator/services/CommunityReconciliationService.mjs` — owns boundary validation of configured reconciliation intervals and admission retry counts while keeping all `AiConfig` defaults `false` / `null` (`AC4`, `AC6`).
    - `learn/agentos/measurements/community-activity-calibration-2026-10-26.md`, `learn/agentos/decisions/0036-durable-community-activity-authority.md`, and `learn/agentos/tooling/CommunitySourceRunbook.md` — record the reproducible calibration report, derived bounds, explicit unset dispositions (`retentionMs: null`, `archiveThreshold: null`, `paginationCap: null`), `E/F/K/O/V/X` (`keep-deferred`) and `J` stage-2 steward lease/wake (`activate-gated` for `#100` under measured bounds while disabled-by-default in config).

---
*Authored by Maryam (Gemini 4 Argon, Antigravity). Session `cd90984f-706d-48de-8f9d-05a76ea2749c`. 🧭*

- 2026-10-26T22:32:08Z @neo-gemini-maryam cross-referenced by PR #1311
- 2026-10-26T22:36:30Z @neo-gemini-maryam referenced in commit `0ae0a32` - "test(memory-core): import config.template.mjs in calibration spec (#3)"
- 2026-10-26T22:40:50Z @neo-gemini-maryam referenced in commit `2a620aa` - "feat(memory-core): calibrate community policy from measured evidence (#3)"
- 2026-10-26T22:40:50Z @neo-gemini-maryam referenced in commit `070639a` - "test(memory-core): import config.template.mjs in calibration spec (#3)"
- 2026-10-26T22:48:38Z @neo-gemini-maryam cross-referenced by PR #1312
- 2026-10-26T23:09:34Z @neo-gemini-maryam cross-referenced by PR #1313
- 2026-10-26T23:17:59Z @neo-gemini-maryam referenced in commit `16152c7` - "fix(memory-core): address review feedback on community policy calibration (#3)"

