---
id: 1953
title: Fleet asks to approve Dependabot-only runs on contributor PRs
state: OPEN
labels:
  - bug
  - ai
  - testing
assignees: []
createdAt: '2026-11-27T07:42:17Z'
updatedAt: '2026-11-27T07:42:17Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1953'
author: neo-gpt-sophie
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
# Fleet asks to approve Dependabot-only runs on contributor PRs

## Context

[Engine #22328](https://github.com/neomjs/neo/pull/22328) received another `approve-runs` wake on unchanged head `0c5b1e25af8dfbea19b6a917a6c5ef2b438ebb7e` after its five ordinary workflows had finished. The remaining held suite is [Dependabot Auto-Merge run 45947116705](https://github.com/neomjs/neo/actions/runs/45947116705).

## The Problem

GitHub’s check-suite read shows exactly one `ACTION_REQUIRED` suite. The other workflows left the approval gate, and [PR Baseline completed with a failure](https://github.com/neomjs/neo/actions/runs/45947116709). The remaining workflow has a Dependabot-only job predicate, while the PR author is a first-time human contributor. As a result, Fleet offers a workflow approval that a maintainer cannot use to advance the PR.

## The Architectural Reality

[The Brain reducer](https://github.com/neomjs/neo-agent-brain/blob/fa7cb4e0fb8d6ff4ef5783a8a22f52b9517b136e/ai/services/fleet/openWorkReducer.mjs#L358-L363) treats all `ACTION_REQUIRED` suites other than review suites as held approvals. [The holder selector](https://github.com/neomjs/neo-agent-brain/blob/dev/ai/services/fleet/openWorkHolder.mjs) gives this result priority over ordinary outside-PR triage. The head-suite query already carries the workflow node identity.

The held workflow is Engine workflow ID `380659107`, node ID `W_kwDODSospM4WsGWj`. Its trusted [job predicate](https://github.com/neomjs/neo/blob/0c1585bcc5f7fc1dac36f272d3f601107e8573ec/.github/workflows/dependabot-automerge.yml#L14-L23) admits only `dependabot[bot]`.

Design authority: #1008 routes a genuine held contributor run to the maintainer rotation. #759 routes a PR to the holder of its next action. #1614 preserves the existing approval permission boundary.

## The Fix

Count only approval-eligible suites when deriving `runApproval`. For this known actor-gated workflow, use trusted repository and workflow identity with the PR author to distinguish an inapplicable run from held contributor CI. Leave the run visible as a GitHub fact. Carry the real remaining next action through the existing outside-PR routing.

## Contract Ledger Matrix

| Surface | Authority | Behavior | Fallback | Evidence |
|---|---|---|---|---|
| `runApproval` and `approve-runs` routing | #759 and #1008 | A known Dependabot-only workflow does not qualify a human contributor’s head for approval. A genuinely held contributor workflow still qualifies. | Incomplete reads retain the existing coverage boundary. | Replay the current head’s suite shape and a mixed held-run fixture. |

## Acceptance Criteria

- [ ] The observed #22328 head, with only this workflow held and ordinary CI completed, emits no `approve-runs` action from that suite.
- [ ] A real held contributor workflow still produces `approve-runs`, including when the excluded suite is also present.
- [ ] Tests verify classification by trusted workflow identity and author, plus the next outside-PR action after ordinary CI finishes.

## Out of Scope

Workflow approval permissions, the Engine merge policy, and the contributor’s PR content review.

Related: #759, #1008, #1614, #1629, #1923.

Decision Record impact: none.

Live latest-open + closed sweep: checked 20 latest open Brain issues, open and closed `approve-runs` hits, and Dependabot overlap. 155 open. The linked issues do not cover this selector case. A2A: latest 30 contains the two wakes and no overlapping claim. Memory, KB and own-assignment sweep found no owner for this correction.

Handoff: @neo-opus-grace, maintainer of the existing outside-run routing contract.

Origin Session ID: 89a7d79e-e6f0-4d9d-ae85-1d47c4bb6822
Retrieval Hint: "approve-runs outside contributor Dependabot Auto-Merge ACTION_REQUIRED repeat #22328"


## Timeline

- 2026-11-27T07:42:19Z @neo-gpt-sophie added the `bug` label
- 2026-11-27T07:42:19Z @neo-gpt-sophie added the `ai` label
- 2026-11-27T07:42:19Z @neo-gpt-sophie added the `testing` label

