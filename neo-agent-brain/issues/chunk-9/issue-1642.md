---
id: 1642
title: 'Outside PR pushed past a seat''s review wakes rotation, not the seat'
state: OPEN
labels:
  - bug
  - contributor-experience
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-12T07:29:35Z'
updatedAt: '2026-11-12T07:29:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1642'
author: neo-gemini-maryam
commentsCount: 0
parentIssue: 759
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
# Outside PR pushed past a seat's review wakes rotation, not the seat

## Context

When an outside contributor pushes a follow-up commit addressing a maintainer seat's `CHANGES_REQUESTED` review, `holderOf` (`ai/services/fleet/openWorkHolder.mjs:35-42`) classifies the new head as `untouched` and wakes a random rotation seat with `[open-work:triage-outside]` (*"an untouched outside contributor pull request has finished CI"*) instead of waking the standing Round-1 reviewer with `[open-work:review]`.

Observed twice live within five minutes on `2026-11-12`:

1. **`neomjs/neo#20801`** (`author_association: "FIRST_TIME_CONTRIBUTOR"`): `@neo-opus-vega` submitted `CHANGES_REQUESTED` ([`5868364233`](https://github.com/neomjs/neo/pull/20801#pullrequestreview-5868364233)) on `c2a17c89` at `03:10:28Z`. When the contributor pushed `87e66d18` and CI turned green, the wake routed to `@neo-opus-ada` via rotation instead of `@neo-opus-vega`, requiring manual A2A relay before `@neo-opus-vega` posted Round-2 `APPROVED` ([`5870481048`](https://github.com/neomjs/neo/pull/20801#pullrequestreview-5870481048)).
2. **`neomjs/neo#20809`** (`author_association: "CONTRIBUTOR"`): `@neo-gemini-julia` submitted `CHANGES_REQUESTED` ([`5870224418`](https://github.com/neomjs/neo/pull/20809#pullrequestreview-5870224418)) on `1401dad5` at `06:40:06Z`. When `@yashh1323` pushed `7146aea9` and CI turned green, `[open-work:triage-outside]` woke `@neo-gemini-maryam` at `07:05:49Z`, requiring manual A2A relay back to `@neo-gemini-julia` for Round-2 `APPROVED` ([`5870580430`](https://github.com/neomjs/neo/pull/20809#pullrequestreview-5870580430)).

## The Problem

`holderOf` models pushed-past change requests around seat-authored PRs (`test/playwright/unit/ai/services/fleet/openWorkHolder.spec.mjs:63-66, 93-95`):

- On a **seat-authored** PR (`owner.kind === 'seat'`), pushing `H2` past `CHANGES_REQUESTED` leaves `requested: []` until CI turns green, wakes the author with `route-review` (`openWorkHolder.mjs:60-62`), and the author seat calls `manage_pr_reviewers` to populate `requested: ['@reviewer']`, which then hits `ci === 'green' && requested.length` (`openWorkHolder.mjs:56-58`).
- On an **outside contributor** PR (`owner.kind === 'outside'`), external contributors (`FIRST_TIME_CONTRIBUTOR`, `CONTRIBUTOR`, `NONE`) lack `WRITE` permission on GitHub and **cannot** re-request a review (`requested` stays `[]` on `H2`), nor do they hold an Agent OS seat to receive `route-review`.
- Meanwhile, `untouched` (`!requested.length && !reviews.some(review => review.onHead && !review.bot)` after PR `#1638`) is strictly head-scoped from `#1008`'s intake note (*"a review on push 1 leaves head 2 untouched"*). Consequently, on `H2`:
  1. **Misrouted green Round 2:** `finished && untouched` matches at `openWorkHolder.mjs:40-42` and emits `{role: 'rotation', kind: 'triage-outside', ids: []}` to a hash-picked seat — even though on GitHub `@reviewer`'s `CHANGES_REQUESTED` opinion (`onHead: false` in `opinions`) remains sticky on `reviewDecision` (another seat's `APPROVED` cannot clear it without dismissal) and `pr-review` Round-2 disposition belongs to the Round-1 reviewer.
  2. **Premature wake on red Round 2:** Because `finished` is `ci === 'red' || ci === 'green'`, if the contributor's Round-2 push `H2` fails CI while they are still iterating, `finished && untouched` also wakes a rotation seat on a red Round-2 commit instead of falling through to `{role: 'author', kind: 'repair-ci', ids: []}`.
  3. **Orthogonal to `#1630`:** `#1630` covers unanswered external prose comments when `headRefOid` does not change (`respond-comment`). When a contributor pushes `H2` without commenting (or once `respond-comment` is answered/coalesced), `holderOf` still needs the head-push rule to route `H2` to the engaged native reviewer rather than `triage-outside`.

## The Architectural Reality

- `Design authority:` *"Terminal predicate: Every open PR on the repositories the Fleet's seats work on wakes the seat that holds its next action once per change of holder (a red head, changes requested, a review due, approval for the human merge, a merge)"* (`#759`); *"The open-work projection and the wake path both read this one function, so they never disagree about whose turn a PR is. An opinion counts only when it judged the current head: `reviewDecision` keeps a change request the author has since pushed past, and an earlier head's approval"* (`ai/services/fleet/openWorkHolder.mjs:3-8`). Amends `#1008`'s head-scoped `untouched` clause for outside PRs that already carry a native `@seat` review on an earlier head.
- **Structure map (`npm run ai:structure-map -- --files --loc`):** Pure holder logic lives in `ai/services/fleet/openWorkHolder.mjs`, consumed identically by `ai/services/fleet/openWorkWakes.mjs:137` and `ai/services/fleet/fleetOpenWorkSource.mjs:56`. Zero new `.mjs` files, zero GraphQL query / `queryHash` changes (`OPEN_WORK_SNAPSHOT` already populates `opinions` and `reviews` with `reviewer`, `state`, `onHead`, and `#1638`'s `bot`), and zero `AiConfig` changes (`ADR 0019`).
- **Planner & ledger compatibility (`ai/services/fleet/openWorkWakes.mjs:177-196`):** `planOpenWorkWakes` already admits `{role: 'reviewer', kind: 'review', ids: ['@seat']}` for any `row.owner.kind` (already exercised when `requested` is non-empty on an outside PR at `openWorkHolder.spec.mjs:21`), keying `ledger.holding` by `[repo, number, head, seat, 'review']`.
- **Sequencing:** Touches `ai/services/fleet/openWorkHolder.mjs:35-58` directly behind open PR `#1638` (`#1629`, `@neo-opus-grace`) and adjacent to `#1630`. Branch from `dev` after `#1638` merges (or fold with `#1630` if one seat takes `#1630` after `#1638`).

## The Fix

In `ai/services/fleet/openWorkHolder.mjs`:

1. **Distinguish genuinely untouched outside PRs from outside PRs already engaged by a native `@seat`:**
   - Identify native `@seat` reviewers from earlier heads (`isSeat = id => typeof id === 'string' && id.startsWith('@') && id.length > 1`):
     - `priorOpinionSeats`: unique native `@seat` reviewers in `opinions` with `!opinion.onHead` who have **not** already reviewed the current head (`!reviews.some(r => r.reviewer === opinion.reviewer && r.onHead)`).
     - `priorCommentSeats`: unique native `@seat` reviewers in `reviews` with `!review.onHead && !review.bot` (fallback when no native `@seat` holds a standing opinion in `opinions`, e.g. a Round-1 `COMMENTED` triage review on `H1`).
     - `engagedOutsideSeats`: `priorOpinionSeats.length ? priorOpinionSeats : priorCommentSeats`.
   - Narrow `untouched` so an outside PR with prior native `@seat` engagement (`engagedOutsideSeats.length > 0` or any non-bot on-head review) is **not** classified as `untouched` for `triage-outside`. ( Note: if only a non-seat human login such as `login:tobiu` reviewed `H1` and zero `@seat` has engaged, `engagedOutsideSeats` is `[]`, so `H2` still reaches `triage-outside` rather than dropping into `ids: []`; `runApproval === 'action_required'` continues to route `approve-runs` regardless of `untouched`.)
2. **Route green pushed-past outside heads to the engaged native reviewer(s):**
   - When `eligible && ci === 'green'` (after the `ci === 'red' || onHead('CHANGES_REQUESTED')`, `incomplete`, and `CONFLICTING` guards) and `!onHead('APPROVED')`:
     - Combine `requested` with any unreviewed-on-head native `CHANGES_REQUESTED` seats from `priorOpinionSeats` (or, when `requested.length === 0`, `engagedOutsideSeats`). If non-empty, return `{role: 'reviewer', kind: 'review', ids}` (deduped in deterministic order).
   - If the engaged `@seat` subsequently posts a `COMMENTED` review on `H2` (`onHead: true` in `reviews` while their `H1` `CHANGES_REQUESTED` remains `onHead: false` in `opinions` under `pr-review` single-RC circuit breaker), `!reviews.some(r => r.reviewer === seat && r.onHead)` becomes `false`, clearing their due-review state on `H2` (`{role: 'none', kind: null, ids: []}`) until the contributor pushes `H3` (or comments per `#1630`).

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback / Edge Case | Docs | Evidence |
|---|---|---|---|---|---|
| `holderOf` outside-PR touch & Round-2 routing (`ai/services/fleet/openWorkHolder.mjs`) | `#759` holder/wake parity; `#1008` (amended); `pr-review-guide.md §6.2–§6.3` | On an `eligible` outside PR pushed to `H2` past a native `@seat`'s review: `H2` is not `untouched` (`ci: 'pending'` → `none`; `ci: 'red'` → `author` `repair-ci` with `ids: []`); when `ci === 'green'`, returns `{role: 'reviewer', kind: 'review', ids}` for prior native opinion seat(s) (preferring `!onHead` `opinions` over `!onHead` `COMMENTED` `reviews`, merged with any explicit `requested` seats) until that seat reviews `H2` | Genuinely unreviewed outside PRs (or bot-only per `#1629`, or non-seat-only prior login) still route `triage-outside`; `runApproval === 'action_required'` still routes `approve-runs`; seat-owned PRs keep `route-review` (`openWorkHolder.spec.mjs:93-95`) | Module & function JSDoc in `openWorkHolder.mjs` | Unit (`openWorkHolder.spec.mjs`, `openWorkWakes.spec.mjs`) |

## Decision Record impact

`none`.

## Acceptance Criteria

- [ ] **AC-1 (Outside PR pushed past native `CHANGES_REQUESTED` or `APPROVED`):** When an `eligible` outside PR has a native `@seat` opinion with `onHead: false` and `requested: []`, `holderOf` returns `{role: 'none', kind: null, ids: []}` while `ci === 'pending'`, `{role: 'author', kind: 'repair-ci', ids: []}` when `ci === 'red'`, and `{role: 'reviewer', kind: 'review', ids: ['@seat']}` when `ci === 'green'` (unit: `openWorkHolder.spec.mjs`).
- [ ] **AC-2 (Opinion precedence + on-head comment clearance + co-reviewer merge):** When Seat A `COMMENTED` on `H1` and Seat B `CHANGES_REQUESTED` on `H1`, green `H2` wakes only `['@seat-b']`; if no opinion exists and Seat A `COMMENTED` on `H1`, green `H2` wakes `['@seat-a']`; once Seat B posts a `COMMENTED` review on `H2` (`onHead: true` in `reviews`, `CHANGES_REQUESTED` still `onHead: false` in `opinions`), `holderOf` returns `{role: 'none', kind: null, ids: []}` on `H2` and re-arms `['@seat-b']` on green `H3`; if Seat C is explicitly `requested` on `H2` alongside Seat B's `!onHead` `CHANGES_REQUESTED`, both `['@seat-c', '@seat-b']` (or preserved order) receive `review` (unit: `openWorkHolder.spec.mjs`).
- [ ] **AC-3 (Planner & seat-PR invariants preserved):** Multi-pulse `planOpenWorkWakes` journey (`H1` untouched `triage-outside` → Seat B `CHANGES_REQUESTED` on `H1` → contributor pushes `H2` green → wakes `@seat-b` with `kind: 'review'`, never `rotation` `triage-outside`); `#1629` bot-only review (`bot: true`), `runApproval: 'action_required'`, non-seat-only prior login (`login:tobiu`), and seat-authored `route-review` (`openWorkHolder.spec.mjs:93-95`) remain intact (unit: `openWorkHolder.spec.mjs`, `openWorkWakes.spec.mjs`).
- [ ] **AC-4 *(post-merge, installed)*:** On the next outside contributor PR pushed past a maintainer's `CHANGES_REQUESTED` review after FM updates, the wake delivers `[open-work:review]` to that Round-1 reviewer rather than `[open-work:triage-outside]` to rotation; record receipt on `#759`.

## Out of Scope

- **Unanswered external comments without a `headRefOid` push:** Owned by `#1630` (`respond-comment`).
- **Bot reviews and baselined initial rotation slots:** Owned by `#1629` / open PR `#1638`.
- **Silent-holder / unreachable-seat escalation:** Owned by `#807`.

## Avoided Traps

- **Changing seat-authored PR routing (`owner.kind === 'seat'`):** Seat authors own `route-review` (`openWorkHolder.mjs:60-62`, `openWorkHolder.spec.mjs:93-95`) so the author posts their `review-response-protocol.md` comment before re-requesting the reviewer; auto-waking `!onHead` reviewers on seat-authored PRs would race the author's response comment.
- **Treating non-rostered GitHub logins (`login:...`) as `engagedOutsideSeats`:** `planOpenWorkWakes` filters `holder.ids` to native `@seat` strings (`openWorkWakes.mjs:123, 179`); suppressing `triage-outside` when zero `@seat` has reviewed would leave `H2` with `ids: []` and wake nobody.
- **Keeping `@seat` in `review` after `@seat` already posted a `COMMENTED` review on `H2`:** Under `pr-review-guide.md §6.3`, a reviewer family cannot post a second `CHANGES_REQUESTED` demand round, so an on-head `COMMENTED` follow-up on `H2` must clear `holderOf` back to `none` until `H3`.

## Related

- Parent: #759
- Sibling open-work leaves: #1008, #761, #779, #1629 (PR #1638), #1630, #807, `neomjs/neo-agent-skills#366`
- Live specimens: `neomjs/neo#20801`, `neomjs/neo#20809`

`good first issue` excluded: touches `ai/services/fleet/openWorkHolder.mjs:35` in active sequence behind open PR `#1638` (`#1629`) and sibling `#1630`, plus post-merge live FM receipt on `#759`.
unowned-rationale: Ready for pickup by an active maintainer seat once PR `#1638` (`#1629`) merges (`@neo-gemini-maryam` is in operator-directed review-only quota reserve).

Live latest-open + closed sweep: checked latest 20 of 158 open issues in `neomjs/neo-agent-brain` (red band >50; attached as sub-issue under owning open Epic `#759`, 18 existing subs < 25 cap; read `#1629`, PR `#1638`, `#1630`, `#1637`, `#1640`, `#1641`) + closed search (`outside contributor CHANGES_REQUESTED triage-outside`, `pushed past review outside`, `triage-outside OR openWorkHolder`; read `#1008`, `#761`, `#779`) at `2026-11-12T07:27:43Z`; no equivalent or prior exclusion found.
A2A in-flight claim sweep: checked latest 30 all-state messages (`2026-11-12T07:27:38Z`); zero competing claim or filing on outside-PR Round-2 holder routing.
MC sweep: `query_raw_memories("outside contributor pull request pushed past CHANGES_REQUESTED triage-outside rotation instead of reviewer")`, 5 results, no prior exclusion found.
Own-assignment sweep: 0 open in `neomjs/neo-agent-brain`, none overlapping.

Origin Session ID: 3fcf378b-7eba-44ca-9bd4-7e185731aace
Retrieval Hint: "outside contributor PR pushed past CHANGES_REQUESTED triage-outside rotation openWorkHolder #20801 #20809"

## Timeline

- 2026-11-12T07:29:37Z @neo-gemini-maryam added the `bug` label
- 2026-11-12T07:29:37Z @neo-gemini-maryam added the `contributor-experience` label
- 2026-11-12T07:29:37Z @neo-gemini-maryam added the `ai` label
- 2026-11-12T07:29:37Z @neo-gemini-maryam added the `agent-os` label
- 2026-11-12T07:30:12Z @neo-gemini-maryam added parent issue #759

