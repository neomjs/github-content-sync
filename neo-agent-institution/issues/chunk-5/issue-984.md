---
id: 984
title: Codex wake setup suggests retrying an incompatible running seat
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gpt-emmy
createdAt: '2026-11-15T12:11:11Z'
updatedAt: '2026-11-15T12:43:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/984'
author: neo-gpt-emmy
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
blocking: []
closedAt: '2026-11-15T12:43:40Z'
---
# Codex wake setup suggests retrying an incompatible running seat

## Context

A Fleet-only update can leave a running Codex seat on its earlier hook generation. A connected Brain can return `nativeWakePreparation: {state: "refused", reason: "codex-wake-compatibility-unproven"}`. The operator can keep working until the next planned managed Start. The [original setup contract](https://github.com/neomjs/neo-agent-institution/issues/980) requires actionable, seat-specific guidance. This is an owned follow-up under #477.

## The Problem

I exercised the production lifecycle adapter at Institution `929e04fa` with that response for an eligible Codex seat. It wrote `unconfirmed`, so the cockpit says “Retry setup”. Repeating setup against the same running generation cannot establish the missing compatibility declaration. A control with `running-seat-changed` wrote `seat-changed`. [Reproduction](https://github.com/neomjs/neo-agent-institution/issues/980#issuecomment-6894069430)

## The Architectural Reality

Design authority: “Keep this chat running. At its next planned managed Start, Fleet stages the installed hook” in the [producer response](https://github.com/neomjs/neo-agent-brain/blob/d6e65d750144c87acf6509f9f7d7065dea46134d/ai/services/fleet/FleetManager.mjs#L728-L736). The adapter maps trusted conditions into a bounded outcome on the selected record. Seat Runtime owns the operator wording. The checked-in Brain pin predates this refusal, while a connected Fleet may already return it.

## The Fix

Recognize the specific Codex compatibility refusal in the existing outcome mapping and present the next planned Start, seat-local hook review if prompted, and ordinary-message verification. Preserve the existing process-start fence and safe fallback.

| Surface | Source | Result | Fallback | Evidence |
| --- | --- | --- | --- | --- |
| Agent Detail / Seat Runtime / native wake setup | Codex refusal reason and selected managed Start | Show how to continue the current chat and when hook setup can proceed | Unknown results keep bounded retry guidance | Production adapter and rendered seat detail with controlled bridge replies |

## Acceptance Criteria

- [ ] A running Codex seat receiving the specific refusal is told to keep its chat running, wait for its next planned managed Start, and review its wake hook in that seat if prompted. It receives neither an immediate retry instruction nor a delivery claim.
- [ ] Other families, other refusal reasons, and stale seat selections keep their existing outcomes. Arbitrary bridge text never appears as operator guidance.
- [ ] A regression test drives the normal intent, reads the feedback on the selected record’s view, and covers the generic fallback.

## Out of Scope

Brain pins, trust changes, lifecycle operations, transport migration and installation. The yellow Codex prompt-field indicator was an operator observation in one build. The fix can use the stable Settings → Hooks route. [Installed evidence](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072691768)

Related: #477, #980, neomjs/neo-agent-brain#1750. Ownership: Emmy is picking this up for the pilot.

Live latest-open + closed sweep: latest 20 open and closed native setup/hook-trust searches on 15 November; 56 open (red). The operator-facing fix is a leaf of the existing recovery-guidance epic. No matching open PR, assigned issue or recent A2A claim. Memory/KB searches on refused setup and retry returned older routing work. Structure map N/A for a change confined to the existing cockpit mapping and view.

Origin Session ID: 01f23f7e-6e0a-4e04-993d-5106dc4bce25

## Timeline

- 2026-11-15T12:11:12Z @neo-gpt-emmy added the `bug` label
- 2026-11-15T12:11:12Z @neo-gpt-emmy added the `agent-os` label
- 2026-11-15T12:11:12Z @neo-gpt-emmy added the `ai` label
- 2026-11-15T12:11:21Z @neo-gpt-emmy added parent issue #477
- 2026-11-15T12:11:23Z @neo-gpt-emmy assigned to @neo-gpt-emmy
- 2026-11-15T12:17:33Z @neo-gpt-emmy cross-referenced by PR #985
- 2026-11-15T12:28:34Z @neo-gpt-emmy referenced in commit `fd89870` - "test(fleet): record verified wake setup visual inputs (#984)"
- 2026-11-15T12:43:40Z @tobiu referenced in commit `408e735` - "fix(fleet): show the next Start for incompatible Codex wake hooks (#984) (#985)

* fix(fleet): show the next Start for incompatible Codex wake hooks (#984)

* test(fleet): record verified wake setup visual inputs (#984)"
- 2026-11-15T12:43:41Z @tobiu closed this issue

