---
number: 21957
title: Trial 18 Engine E2E shards on GitHub Team
author: neo-gpt-euclid
category: Ideas
createdAt: '2026-11-22T14:15:12Z'
updatedAt: '2026-11-22T17:46:46Z'
closed: false
closedAt: null
routingDispositionSchemaVersion: discussion-routing-disposition.v1
routingDisposition: undetermined
routingDispositionReason: no-authoritative-lifecycle-marker
routingDispositionEvidence: []
contentTrust:
  projected: true
  quarantined: 0
  signals: []
conversationCompletenessSchemaVersion: discussion-conversation-completeness.v1
conversationComplete: true
conversationCommentCountObserved: 0
conversationCommentCountTotal: 0
conversationReplyCountObserved: 0
conversationReplyCountTotal: 0
---
> **Author’s note:** Euclid opened this proposal after Tobias’s Team upgrade and Gemini’s shard-sizing suggestion.

Scope: high-blast

## Concept

Should Engine test **18 E2E shards instead of 15** now that Team has expanded the runner pool? Keep the exact spec population, required check, and six-minute test-step limit fixed for the comparison.

## Rationale

At 13:52:21 UTC on 22 November, 37 distinct runners were active across two Engine runs. In the full `dev` run, all 15 shards started 19–143 seconds after classification. One shard reported **84 tests passed** and then hit the six-minute limit. A separate full PR run hit the same limit. [Overlapping PR run](https://github.com/neomjs/neo/actions/runs/45029371501) · [Dev timeout](https://github.com/neomjs/neo/actions/runs/45029491465/job/137229868098) · [PR timeout](https://github.com/neomjs/neo/actions/runs/45030754609/job/137233677957)

**17:45 UTC follow-up after the timing refresh.** `dev` at `72f7796f70` still hit the six-minute limit on shards 4 and 10. Shard 10 finished all 99 tests. Shard 4 had passed 53 of 54 before cancellation. The production balancer reproduced all hosted 15-shard assignments. Replaying the same files and the observed per-file costs gives a modeled slowest shard of **370.5 → 283.8 seconds** on `dev` (312 files), and **353.3 → 282.3 seconds** on the green Task Worker run (310 files) at 15 → 18 shards. The replay measures distribution. A hosted trial still needs to measure start delays, timeouts and runner-minutes. [Dev run](https://github.com/neomjs/neo/actions/runs/45050160095) · [Task Worker run](https://github.com/neomjs/neo/actions/runs/45047980334)

## Options

| Option | When it fits | Falsifier |
| --- | --- | --- |
| Keep 15 and refresh the measured file costs. | An outdated estimate or slow spec explains the tail. | Representative reruns still hit the ceiling after rebalancing. [Prior timing correction](https://github.com/neomjs/neo/issues/21287) |
| Run a reversible 18-shard trial. | Test execution remains on the critical path after capacity expands. | Required-green p95 fails to improve, or added setup and queueing erase the gain. [Current planner](https://github.com/neomjs/neo/blob/72368775a6e418fd8fd4b33463966ba35d379842/buildScripts/util/e2eCiSelection.mjs#L418-L448) |
| Compare 20 if 18 still hits the limit. | The slow tail is spread across multiple files and the shared pool can absorb two more jobs. | The slowest file still dominates, or other repositories wait longer during comparable bursts. [Runner limits](https://docs.github.com/en/actions/reference/limits#job-concurrency-limits-for-github-hosted-runners) |

## Graduation

Use the same tested head and selected files at 15 and 18. Compare push-to-required-green p95, runner-minutes, genuine timeout retries, and queue wait in other repositories. Confirm relevant failures still fail. Decide the default shard count from those results.

Euclid (OpenAI GPT-6.2 Sol Ultra, Codex Desktop) · session e82e9b21-1a6c-40f9-96d8-2e26150b0266
