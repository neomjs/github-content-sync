---
id: 908
title: Memories reader accepts decisionContext alongside legacy thought
state: CLOSED
labels:
  - bug
  - agent-os
  - ai
assignees:
  - neo-gemini-julia
createdAt: '2026-11-10T23:36:55Z'
updatedAt: '2026-11-11T02:10:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/908'
author: neo-gemini-julia
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
blocking:
  - '[ ] 910 Retire the legacy thought alias after the Brain pin moves'
closedAt: '2026-11-11T02:10:35Z'
---
# Memories reader accepts decisionContext alongside legacy thought

## Context

`neomjs/neo-agent-brain#1580` (PR `neomjs/neo-agent-brain#1582`) completes the read-side cutover from `thought` to `decisionContext` across Memory Core MCP read surfaces, including `get_session_memories`. In the Brain's Fleet adapter (`ai/services/fleet/fleetSessionMemoriesSource.mjs`), `get_session_memories` rows pass through directly to the Fleet Manager cockpit. Today, the Fleet Manager's Memories drill-in reads only the legacy `thought` key (`apps/agentos/model/SessionTurn.mjs:57` and `apps/agentos/view/fleet/memories/ReaderComponent.mjs:8`), so once the plane cuts over to `decisionContext`, every turn in the Memories reader silently loses its middle block (`convert` coerces `undefined` to `null` and renders the missing-field notice instead of the turn's decision context).

- Live latest-open + closed sweep: checked latest 20 open (`50` open total, orange band; `#907`..`#635`) + closed search (`Memories reader SessionTurn thought decisionContext`) at `2026-11-10T23:33:17Z`; no equivalent or prior exclusion found
- A2A in-flight claim sweep: checked latest 30 messages across all read states (`status: 'all'`) at `2026-11-10T23:33:27Z`; no overlapping `[lane-claim]` or `[lane-intent]`
- MC sweep: `"Memories reader loses middle block SessionTurn ReaderComponent thought get_session_memories"`, 5 results + KB ticket sweep (5 hits), no prior decision found
- Own-assignment sweep: 0 open in `neomjs/neo-agent-institution`, none overlapping

## The Problem

1. `apps/agentos/model/SessionTurn.mjs:57-61` defines the middle authored turn field solely as `thought` (`convert: value => typeof value === 'string' ? value : null`).
2. `apps/agentos/view/fleet/memories/ReaderComponent.mjs:8` declares `const TURN_FIELDS = [['prompt', 'Prompt'], ['thought', 'Thought'], ['response', 'Response']]`.
3. After `neomjs/neo-agent-brain#1582` merges, `get_session_memories` returns `{id, sessionId, timestamp, prompt, decisionContext, response, agentIdentity, amountToolCalls, ...}` with no `thought` key.
4. Without a dual-key bridge in `neo-agent-institution`, any Fleet Manager build talking to a post-`#1582` plane coerces the missing `thought` property to `null` and omits the middle block on every turn without throwing an error, while an immediate single-key cutover to `decisionContext` alone would break an updated Fleet Manager talking to a pre-`#1582` plane.

## The Architectural Reality

Design authority: `"One view-projection record in the Fleet memories drill-in: a single turn-level memory as the get_session_memories operation returned it."` (`apps/agentos/model/SessionTurn.mjs:7-8`) and `"The wire already carries each record whole — get_all_summaries the summary document, get_session_memories the turn's prompt, thought and response — so nothing is fetched here and nothing is cut"` (`apps/agentos/view/fleet/memories/ReaderComponent.mjs:16-18`), paired with `neomjs/neo-agent-brain#1580` / `neomjs/neo-agent-brain#1550` (`decisionContext` replaces `thought` across Memory Core write and read surfaces).

- `apps/agentos/model/SessionTurn.mjs` is the canonical `Neo.data.Model` boundary normalizing raw `get_session_memories` wire bags before any Memories list or reader component consumes them.
- Normalizing `decisionContext` as the canonical model field (`mapping: 'thought'`, falling back to `record.thought` when `decisionContext` is absent on a pre-cut plane) lets `ReaderComponent.mjs` render `['decisionContext', 'Decision Context']` via `record[field]` cleanly across both pre-cut (`thought`) and post-cut (`decisionContext`) plane deployments with zero deploy-order coupling, because `ReadingController#syncReader` feeds the reader solely from `RowsGrid#recordBag` or the Store's `SessionTurn` records.
- Check sibling Memories view/spec files under `apps/agentos/view/fleet/memories/` and `test/playwright/unit/` for any fixtures or assertions referencing `SessionTurn` `thought` or `Thought` block headings so unit coverage exercises both wire shapes.

## The Fix

1. Update `apps/agentos/model/SessionTurn.mjs` so the middle authored field is `decisionContext` (`mapping: 'thought'`, string-guarded `convert` defaulting to `null`), populated from `decisionContext` when present on the wire row and falling back to legacy `thought` when the row carries no `decisionContext` key (deploy-order independent; a non-string `decisionContext` still converts to `null` per the vocabulary-collision guard).
2. Update `apps/agentos/view/fleet/memories/ReaderComponent.mjs` (`TURN_FIELDS`, lowercase block copy/missing nouns, and JSDoc) to read `['decisionContext', 'Decision Context']` directly via `record[field]` with no second normalization site, and align `apps/agentos/view/fleet/memories/Container.mjs` drill-head authored provenance copy (`prompt · decision context · response`).
3. Update existing Memories unit specs and add focused unit assertions verifying `SessionTurn` + `ReaderComponent` render and copy the middle block identically through the Store whether the upstream `get_session_memories` row supplies `decisionContext` (post-`brain#1582`) or legacy `thought` (pre-`brain#1582`), while non-string values still convert to `null`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `AgentOS.model.SessionTurn` (`apps/agentos/model/SessionTurn.mjs:57-61`) | `neomjs/neo-agent-brain#1580`, `SessionTurn.mjs:7-13` | Exposes `decisionContext` (`String\|null`, `mapping: 'thought'`), accepting wire `decisionContext` first and legacy `thought` when the row lacks `decisionContext` | Non-string values convert to `null` (existing vocabulary-collision guard) | JSDoc on `SessionTurn` | Playwright unit spec |
| `AgentOS.view.fleet.memories.ReaderComponent` (`apps/agentos/view/fleet/memories/ReaderComponent.mjs:8`) | `neomjs/neo-agent-brain#1580`, `ReaderComponent.mjs:13-20`, `ReadingController.mjs:158-167` | Renders `['decisionContext', 'Decision Context']` as the middle turn block via `record[field]` from `RowsGrid#recordBag` projections and Store `SessionTurn` records | Missing/non-string field named with lowercase block label and record id; no reader-side `thought` alias | JSDoc on `ReaderComponent` | Playwright unit spec |
| `AgentOS.view.fleet.memories.Container` (`apps/agentos/view/fleet/memories/Container.mjs:197,203`) | `neomjs/neo-agent-brain#1580`, `Container.mjs:38-40` | Updates drill-head authored provenance tooltip and legend copy from `prompt · thought · response` to `prompt · decision context · response` | None (static chrome copy) | JSDoc on `MemoriesPane` | Playwright unit + NL E2E specs |

## Decision Record impact

`none`

## Acceptance Criteria

- [ ] `AgentOS.model.SessionTurn` normalizes both `{decisionContext: '...'}` (post-`brain#1582`) and `{thought: '...'}` (pre-`brain#1582`) wire rows onto `decisionContext`, while converting non-string values to `null`.
- [ ] `AgentOS.view.fleet.memories.ReaderComponent` renders the middle turn section (`Decision Context`) for both `decisionContext` and legacy `thought` turn rows (single-turn `reading.record` and whole-store `*show all*` register) without requiring coordinated Brain/Institution deploy order.
- [ ] Unit tests under `test/playwright/unit/` pass and explicitly cover `decisionContext`-only rows, legacy `thought`-only rows, and non-string/missing field handling.
- [ ] *(Post-merge live check, paired with `neomjs/neo-agent-brain#1582`)* Open the Fleet Manager Memories reader against a live plane running `neomjs/neo-agent-brain#1582` and verify turn records display all three blocks (`Prompt`, `Decision Context`, `Response`).

## Out of Scope

- Changing `get_all_summaries` session-summary rendering.
- Modifying Brain-side `fleetSessionMemoriesSource.mjs` (tracked in `neomjs/neo-agent-brain#1580` / PR `#1582`).

## Avoided Traps

- **Hard-cutting `SessionTurn` to `decisionContext` without legacy `thought` fallback:** Packaged or local Fleet Manager builds can run ahead of or behind the containerized Memory Core plane; accepting `decisionContext` first and falling back via `mapping: 'thought'` when the row lacks `decisionContext` makes deploy order irrelevant.
- **Using `record.decisionContext || record.thought` instead of `mapping: 'thought'` + string guard:** An empty string `""` or malformed object `{}` on `decisionContext` should be governed by `decisionContext`'s own string guard, not silently fall through to a stale `thought` property.

## Related

- `neomjs/neo-agent-brain#1580` / PR `neomjs/neo-agent-brain#1582` (producer-side `get_session_memories` `decisionContext` cutover; review finding `PRR_kwDOUBzDFM8AAAABXL8qLg` RA-1 by `@neo-fable-clio`)
- `#910` (retirement of the legacy `thought` alias once the pinned Brain commit moves past `4f3b8fbb`)
- `#505` (Cockpit views epic)

Ownership: unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: 3b4d3f8f-c078-4cee-8876-b0db074a5c04
Retrieval Hint: "SessionTurn ReaderComponent Memories reader decisionContext legacy thought get_session_memories fleetSessionMemoriesSource"

## Timeline

- 2026-11-10T23:36:56Z @neo-gemini-julia added the `bug` label
- 2026-11-10T23:36:57Z @neo-gemini-julia added the `good first issue` label
- 2026-11-10T23:36:57Z @neo-gemini-julia added the `help wanted` label
- 2026-11-10T23:36:57Z @neo-gemini-julia added the `agent-os` label
- 2026-11-10T23:36:57Z @neo-gemini-julia added the `ai` label
- 2026-11-10T23:55:17Z @neo-gemini-julia cross-referenced by #1580
- 2026-11-10T23:55:42Z @neo-gemini-julia cross-referenced by PR #1582
- 2026-11-11T00:29:33Z @neo-gemini-julia assigned to @neo-gemini-julia
- 2026-11-11T00:32:56Z @neo-gemini-julia removed the `good first issue` label
- 2026-11-11T00:32:56Z @neo-gemini-julia removed the `help wanted` label
- 2026-11-11T00:49:40Z @neo-gemini-julia cross-referenced by PR #909
- 2026-11-11T00:52:42Z @neo-gemini-julia referenced in commit `1c7a00e` - "docs(memories): remove ticket archaeology from SessionTurn field JSDoc (#908)"
- 2026-11-11T01:23:54Z @neo-fable-clio cross-referenced by #910
- 2026-11-11T01:24:03Z @neo-fable-clio marked this issue as blocking #910
- 2026-11-11T01:30:35Z @neo-gemini-julia referenced in commit `b8fdb34` - "fix(memories): keep legacy thought normalization solely on SessionTurn (#908)"
- 2026-11-11T02:10:35Z @tobiu referenced in commit `e823ea8` - "feat(memories): accept decisionContext alongside legacy thought in Memories reader (#908) (#909)

* feat(memories): accept decisionContext alongside legacy thought in Memories reader (#908)

* docs(memories): remove ticket archaeology from SessionTurn field JSDoc (#908)

* fix(memories): keep legacy thought normalization solely on SessionTurn (#908)"
- 2026-11-11T02:10:36Z @tobiu closed this issue

