---
id: 477
title: Require a native Neo usage check before prescribing Engine work
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-26T22:54:34Z'
updatedAt: '2026-11-26T23:44:37Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/477'
author: neo-gpt-emmy
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
# Require a native Neo usage check before prescribing Engine work

## Context

In the [November 26 audit of 15 Engine tickets](https://github.com/neomjs/neo/issues/22237#issuecomment-7204965150), my form brief directed a peer to implement an advanced state pattern before I tried `Neo.form.Container`. My command-palette brief accepted an example viewport that another application could not import. My disabled-field brief omitted the working global pointer rule and the positive control for application updates.

## The Problem

A ticket can cite a related source file and pass today’s prescription check without establishing what an app author can already accomplish through public Neo APIs. Implementation and review can then reinforce the same mistaken premise.

## The Architectural Reality

The shared [ticket-create workflow §2](https://github.com/neomjs/neo-agent-skills/blob/5431cdadb00b/.agents/skills/ticket-create/references/ticket-create-workflow.md#L102-L114) already asks authors to challenge the prescription and owner. Both plugin and maintainer paths consume it. The [existing design-authority rule](https://github.com/neomjs/neo-agent-skills/blob/5431cdadb00b/.agents/skills/ticket-create/references/design-authority.md) establishes intended behavior. #191 owns the tests/reviewer question. This issue concerns discovery and evidence *before filing*.

Design authority: "Start with the matching engine primitive; class suffix names its base family."

The [Engine app-authoring rule](https://github.com/neomjs/neo-agent-skills/blob/5431cdadb00b/agents-md/sections/1600-edge-case-triggers.md#L14) states this for app code. Tobi's November 26 direction extends the same investigation to the ticket that prescribes Engine, example or Learn work. Decision Record impact: none.

## The Fix

Strengthen the existing authoring step. Make authors investigate an ordinary Neo usage path, verify its owner and a real consumer, and demonstrate the residual gap before prescribing code or teaching a pattern. Route a genuine teaching gap to a guide or example. For a missing public capability, specify how an independent application would consume it and keep domain wiring with the example. Put the finding and its consequence in the ticket’s existing sections.

## Acceptance Criteria

- [ ] Both maintainer and standalone/plugin paths require the check for Neo Engine feature, defect, example and learning tickets. A standalone author can use repository inventory, source and shipped consumers without private tools.
- [ ] A real positive control and an observation through the normal entry point can narrow or eliminate a proposed fix. Examples include ordinary form submission and intended positive-duration Grid inertia. A genuine end-to-end example remains a valid result.
- [ ] A reusable control requires an independently usable public entry rather than copying a demo viewport. Verify plugin parity and package checks. No new issue template field or PR-review ceremony.

Live latest-open + closed sweep: checked the latest 20, open problem owners and closed ticket-create/premise precedents. #6 owns incident admission, #191 the test/reviewer oracle and #439 the in-practice PR question. Own open #20 concerns self-repair. Inventory and the audit are the public evidence. Open backlog 44 (orange). I am taking this operator-requested change.

Retrieval Hint: Neo engine-native capability inventory, app adoption, reusable controls and real user behavior before ticket handoff.
Origin Session ID: 2eda18ec-eae4-46ad-849e-9c72b749e2e6

## Timeline

- 2026-11-26T22:54:34Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-26T22:54:36Z @neo-gpt-emmy added the `enhancement` label
- 2026-11-26T22:54:36Z @neo-gpt-emmy added the `ai` label
- 2026-11-26T22:54:36Z @neo-gpt-emmy added the `model-experience` label
- 2026-11-26T22:54:45Z @github-actions added the `needs-design-authority` label
- 2026-11-26T23:00:03Z @neo-gpt-emmy cross-referenced by PR #478
- 2026-11-26T23:44:37Z @github-actions removed the `needs-design-authority` label
- 2026-11-26T23:47:03Z @neo-gpt-emmy referenced in commit `f7729d7` - "docs(ticket-create): name the Engine inventory and clarify the exercise (#477)"

