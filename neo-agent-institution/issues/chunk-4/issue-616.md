---
id: 616
title: 'Start shows its live preparation on the card, Cancel start and Skip'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
  - design
assignees:
  - neo-opus-grace
createdAt: '2026-10-09T03:33:28Z'
updatedAt: '2026-10-09T06:17:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/616'
author: neo-opus-grace
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
# Start shows its live preparation on the card, Cancel start and Skip

## Context

Split from #610 so that its first half can close on its own. The open #610 PR delivers #610's AC-1 and AC-4: each checkout's preparation in Agent Detail's Repository pane, and the card's `skills not verified` exception. This leaf carries #610's AC-2 (Skip) and AC-3 (the card names the install while it runs). Their producers are neomjs/neo-agent-brain#938, merged and pinned at `03da5025`, and neomjs/neo-agent-brain#942, the Skip verb, still open as neomjs/neo-agent-brain#943.

Tonight's moves show what is missing. Emmy's and Mnemosyne's first Fleet Starts both ran past the card's 30 s answer window while their installs kept going. Each card read `start… no answer yet` until the late answer settled, see the receipts on neomjs/neo-agent-brain#571 ([6072044036](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072044036), [6072521724](https://github.com/neomjs/neo-agent-brain/issues/571#issuecomment-6072521724)).

## The Problem

While a Start installs, the card says only `start…`, then `start… no answer yet`, even though the Fleet already reports each checkout's live row. The operator cannot skip a slow install. While any action is pending the card's toggle is disabled, so Stop is unreachable for the whole install.

## The Architectural Reality

- **Producer:** `fleetCockpitStatus` `dependencyOutcomes`, mapped to `FleetAgent.dependencyOutcomes` by the #610 PR. `installing` appears only while the Start is pending, and undecided rows are left out (`installSeatDependencies`, Brain `03da5025`).
- **Skip verb:** `skipAgentDependencies(agentId)` returns `{id, skippedStarts}` (neomjs/neo-agent-brain#942). An older Fleet refuses the unknown method.
- **Card:** in `roster/card/Container.mjs` `applyRecord()`, the control-status priority puts `pendingAction` first. The toggle is `disabled` whenever an action is pending, and its `iconCls` and `aria-label` follow the old runtime state.
- **Adapter:** `FleetLifecycleIntentAdapter` calls `bridge.start(agentId)`. #608 (merged in #609) reconciles late answers.

## The Fix

The design is settled on #610: [the read](https://github.com/neomjs/neo-agent-institution/issues/610#issuecomment-6067909778), [Sophie's bounds](https://github.com/neomjs/neo-agent-institution/issues/610#issuecomment-6068037141), [adopted](https://github.com/neomjs/neo-agent-institution/issues/610#issuecomment-6068056433).

1. A confirmed live preparation phase refines the pending Start's label: `start… preparing dependencies (n/3 done)`. It replaces the generic `start…` and the local `no answer yet`. A retained phase from another attempt never makes an unanswered Start look live, and an active Stop still reads stopping.
2. During a confirmed preparation the toggle becomes **Cancel start**, with its icon, accessible label and emitted Stop intent in agreement. Once the operator cancels, the card reads canceling until the Start settles, and a late Start reply never clears that newer state.
3. While the phase is live, the Repository pane offers **Skip remaining preparation**, with its consequence beside it: the launch continues, and skills may be unverified while the working checkout is unfinished. Skip binds to the displayed seat and the current attempt, and Stop wins over a concurrent Skip. An older Fleet's refusal reads as Skip unavailable.

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / Edge case | Docs | Evidence |
|---|---|---|---|---|---|
| Card control status (`CARD-CONTRACT.md` Control status) | `dependencyOutcomes` rows with `installing` while the Start is pending | `start… preparing dependencies (n/3 done)`; the title names each live row | no live row, or one from another attempt → today's pending/timeout text | CARD-CONTRACT row | unit + visual |
| Card toggle | the same live phase | Cancel start: icon, `aria-label` and Stop intent agree | no confirmed phase → today's disabled toggle | CARD-CONTRACT Control verbs | unit |
| Repository pane Skip | `skipAgentDependencies(agentId)` (neomjs/neo-agent-brain#942) | Skip remaining preparation, offered only while live | method refused → Skip unavailable | JSDoc | unit + visual |

## Acceptance Criteria

- [ ] AC-1 (#610 AC-3): while a Start installs, the card names the preparation with a repository-completion count, replacing `start…` and the timeout text; the Repository pane names each live row.
- [ ] AC-2 (#610 AC-2): the operator can Skip the remaining install from the Repository pane. Interrupted checkouts read `skipped`, finished ones keep their outcome, and the seat launches. *#629 delivers the pane's Skip against a wire with the verb (unit, fixture goldens). Deferred, owned by #633: the pin carrying neomjs/neo-agent-brain#942, and the installed Skip witness (receipt on neomjs/neo-agent-brain#571).*
- [ ] AC-3: during a confirmed preparation the toggle is Cancel start; a Stop during the install leaves canceled rows and no launch. *#629 delivers Cancel start and its open cancel through the Fleet's answer (unit). Deferred, owned by #633: the installed cancel witness, canceled rows and no launch (receipt on neomjs/neo-agent-brain#571).*
- [ ] AC-4: unit and visual coverage for pending → preparation → late settlement, loss of live progress, Skip after one checkout finished, Stop racing Skip, and a Fleet without the Skip verb.

## Out of Scope

- The Skip verb (neomjs/neo-agent-brain#942) and the install itself (neomjs/neo-agent-brain#937).
- The readiness exception and the per-checkout rows (#610).

## Related

#610 · #608 · neomjs/neo-agent-brain#937 · neomjs/neo-agent-brain#938 · neomjs/neo-agent-brain#942 · neomjs/neo-agent-brain#943 · neomjs/neo-agent-brain#571

Decision Record impact: none.

Live latest-open sweep: the latest 20 open Institution issues at 03:32Z and again at 03:33Z; no equivalent beyond #610 itself.
A2A claim sweep: last 30 messages; no claim on these surfaces (neomjs/neo-agent-brain#942 is the Brain verb).
MC sweep: no decision beyond #610's settled design and the 2026-09-30 operator decision on #245.
Own-assignment sweep: #610 (the parent of this split), #490, #414, #11; no other overlap.

Origin Session ID: e76b2469-377c-4fec-85a7-4c47b10269b9
Retrieval Hint: "Start live dependency preparation card progress Cancel start Skip remaining preparation"


## Timeline

- 2026-10-09T03:33:29Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-09T03:33:30Z @neo-opus-grace added the `enhancement` label
- 2026-10-09T03:33:30Z @neo-opus-grace added the `agent-os` label
- 2026-10-09T03:33:30Z @neo-opus-grace added the `ai` label
- 2026-10-09T03:33:30Z @neo-opus-grace added the `design` label
- 2026-10-09T03:34:56Z @neo-opus-grace cross-referenced by PR #617
- 2026-10-09T03:35:12Z @neo-opus-grace cross-referenced by #610
- 2026-10-09T03:45:36Z @neo-fable-clio cross-referenced by #618
- 2026-10-09T03:56:50Z @neo-fable cross-referenced by #620
- 2026-10-09T03:57:19Z @neo-opus-vega cross-referenced by #942
- 2026-10-09T03:57:21Z @neo-opus-vega cross-referenced by PR #943
- 2026-10-09T05:14:34Z @neo-opus-grace cross-referenced by PR #629
- 2026-10-09T05:45:27Z @neo-opus-grace referenced in commit `8f03d7b` - "chore(agentos): merge the #610 branch's review round into the #616 branch (#616)

# Conflicts:
#	apps/agentos/view/fleet/detail/Container.mjs
#	test/playwright/visual/__screenshots__/baseline-inputs.txt"
- 2026-10-09T06:17:19Z @neo-opus-grace cross-referenced by #633
- 2026-10-09T06:22:52Z @neo-opus-grace referenced in commit `7998ffd` - "fix(agentos): a Skip binds to the attempt it was asked in, and an unanswered cancel stays canceling (#616)"
- 2026-10-09T06:22:52Z @neo-opus-grace referenced in commit `21e56b8` - "test(agentos): the earlier attempt's late Skip answer never replaces the current request (#616)"
- 2026-10-09T06:22:53Z @neo-opus-grace referenced in commit `7f5311e` - "test(agentos): re-stamp the visual baselines over the attempt-bound Skip and the open cancel (#616)"
- 2026-10-09T06:36:59Z @neo-opus-grace cross-referenced by #635
- 2026-10-09T06:53:27Z @neo-opus-grace cross-referenced by #638

