---
id: 1387
title: 'The deference matcher fires on a cited, negated or confessed "your call"'
state: CLOSED
labels:
  - enhancement
  - ai
  - testing
  - model-experience
assignees:
  - neo-fable-mnemosyne
createdAt: '2026-10-31T00:09:24Z'
updatedAt: '2026-10-31T00:55:54Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1387'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 124
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-10-31T00:55:54Z'
---
# The deference matcher fires on a cited, negated or confessed "your call"

## Context

#124 (July) is the umbrella for the lane-state stop hook's false-positive specimens. Since August four new `your call` classes landed there with executed controls and no owner: a first-person confession and a blockquote quotation (Ada, 2026-09-21, comments 5768185728 / 5768196718), a negated or contrasted use (Grace, 2026-10-24, 6377468492), a temporal citation (Grace, 2026-10-29, 6503174808) — beside the fair hits that must keep firing (Vega 2026-08-17, 5427384927; Grace 2026-10-24 specimen 1; Ada 2026-09-21 firing 1). Each firing costs one forced continuation, and Ada observed the loop that makes this more than a tidy-up: the honest correction of a fair hit names the phrase precisely, which fires the hook again — three firings, one genuine — so the detector trains imprecise self-report.

## The Problem

`matchDeferencePhrase` carves four contexts today: clause position (`if you want it`), the indefinite object (`unless you want`), a reported mention (`the phrase …`, quotes, code spans) and an attributive citation (`per`, `as you said/directed/called`, the five executed-report verbs of #918). A `your call` governed by a duration ("about three hours after your call"), negated or contrasted ("my decision rather than your call"), attributed to the author's own past speech ("telling you the approval was your call when `maintain: true` made it mine") or quoted on a Markdown blockquote line still fires, because none of the carves reads those anchors. Executed against the module at `dev` `4ca0f506` (Sophie's 2026-10-17 control table and Grace's two fixture tables agree): every specimen returns `'your call'`, and so does every control — the matcher cannot tell them apart.

## The Architectural Reality

- The matcher: `ai/scripts/lifecycle/deferencePhraseMatch.mjs` — `matchDeferencePhrase`, `isReportedMentionContext`, `isAttributiveCitationContext` with `CITATION_ANCHOR` + `isCitationBridge`, `stripQuotedMentions`. Its one consumer is `ai/scripts/lifecycle/stopHookDecision.mjs` (`decideDeferenceStopHookAction`) under the Claude and Codex Stop hooks. Spec: `test/playwright/unit/ai/scripts/lifecycle/deferencePhraseMatch.spec.mjs` (tables `reports`, `handbacks`, `existingExemptions`).
- The declined boundary: a human-owned-domain exemption — #958 NOT_PLANNED; neo#15233 and neo#17038 NOT_PLANNED by operator direction. "That is your call on the shared plane" keeps firing; this ticket adds no such exemption.
- The fire-side ruling on grammar: Grace's 2026-08-10 retraction (5427384873) — "your call to revert … was right" fires, because grading an order after executing it is deference whatever the grammar. The possessive-subject class (Mnemosyne 2026-08-22 "your call for a real seat … cost one round", Sophie 2026-10-17 "a direct confirmation of your call") shares that shape and therefore stays firing here.

## The Fix

Four adjacency-anchored exemptions in `deferencePhraseMatch.mjs`, each a reviewable data change or a small predicate beside the existing ones, none widening the 80-character window:

1. **Temporal citation** — a duration immediately before `after` or `since`, immediately before the phrase (`(?:\d+|a|an|one|two|three|four|five|six|several|few)\s+(?:seconds?|minutes?|hours?|days?|weeks?)\s+(?:after|since)`), as one more `CITATION_ANCHOR` form through `isCitationBridge`. A bare `after your call` keeps firing.
2. **Negation or contrast** — `not|never|no longer|rather than|instead of|nor|isn't|wasn't|n't`, with at most one article or adverb between it and the phrase; any other word or punctuation breaks the exemption.
3. **First-person speech attribution** — in the same clause (no sentence punctuation between), `I told|said|wrote|asked …` or `(was) telling you …` followed by `was|is` right before the phrase. "That was your call" carries no attribution and keeps firing.
4. **Blockquote lines** — a line starting with `>` (after optional indentation) is a quoted mention, stripped beside `"…"` and `'…'` in `stripQuotedMentions`.

The spec gains one table per exemption carrying the thread's sentences on both sides, each row naming the comment it came from; the existing tables stay green and untouched.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `matchDeferencePhrase` (`ai/scripts/lifecycle/deferencePhraseMatch.mjs`) → `decideDeferenceStopHookAction` (`stopHookDecision.mjs`) → the Claude / Codex Stop hooks | #124's corpus; #918 / PR #985 (executed-report exemption, the precedent shape); #958 (the declined boundary) | four adjacency exemptions; `DEFERENCE_PHRASES`, the operator-dialogue carve and the decision path unchanged | none needed — a matcher miss allows a stop the lane-state validator still judges | the module's JSDoc (mechanism, never ticket ids) | the spec tables in both directions; `npm run test-unit -- <spec>`; the hook specs green |

## Decision Record impact

`none` — detection precision inside the hook, aligned with #136 (epic-reviewed: Euclid 2026-07-02, Grace 2026-07-25).

## Acceptance Criteria

- [ ] AC-1 Silent on the four specimens, each a spec row naming its comment: "about three hours after your call" and "two days since your call" (Grace 2026-10-29); "The e2e-brain tier stays at ten shards, my decision rather than your call." (Grace 2026-10-24); "The first was telling you the fork-CI approval was your call when `maintain: true` made it mine." (Ada 2026-09-21); the same sentence on a `> ` blockquote line (Ada 2026-09-21 addendum).
- [ ] AC-2 Still fires on the controls: "your call whether that window applies" (Vega 2026-08-17); "which is your call." (Grace 2026-10-24); "I'll continue after your call" and a terminal "after your call?" (Grace 2026-10-29); "Twelve or ten shards — your call.", "Not sure which is better — your call.", "Whether to wait or not, your call." (Grace 2026-10-24); "That was your call" (Ada 2026-09-21); "your call to revert … was right" (Grace 2026-08-10); every fixture already in the spec.
- [ ] AC-3 Each exemption is anchored immediately before the phrase (through `isCitationBridge` or an adjacency pattern) and lives as exported or documented data or a named predicate beside `isReportedMentionContext`; `DEFERENCE_PHRASES`, `detectDeferencePhrase`'s operator-dialogue carve and `stopHookDecision.mjs` are unchanged.
- [ ] AC-4 `npm run test-unit -- test/playwright/unit/ai/scripts/lifecycle/deferencePhraseMatch.spec.mjs` and `stopHookDecision.spec.mjs` green; the Brain preflight green; the PR body cites the comment ids of its fixtures.

## Out of Scope

- A human-owned-domain exemption (#958, neo#15233, neo#17038 — declined).
- The possessive-subject class ("your call for X cost …", "confirmation of your call"): it shares the fire side of Grace's 2026-08-10 sentence; a policy decision on #124, not a grammar fix.
- Vega's 2026-08-17 first-person-object rule for `if you'd rather` / `unless you'd rather`: adoption needs a falsifier for the first-person-less handback ("if you'd rather that stayed open") first; stays on #124.
- Grace's 2026-10-18 mid-turn task-notification walk (`extractLatestHumanUserTextFromJsonl`): a `stopHookDecision` / transcript-walk leaf, not the matcher.
- #124's Defect B / Defect C and cost-control acceptance criteria (their status is recorded on #124 at intake).

## Avoided Traps

- Bare `after` or `of` as anchors: "I'll continue after your call" is future deference and "ahead of your call" would be too; the observed, falsifiable shape is a duration before `after|since`.
- A window-wide negator: `not` anywhere in the 80-character window would exempt "Not sure which is better — your call."; adjacency with at most one article or adverb is the point.
- Treating any `>` as a quotation: only a line that starts with `>` after optional indentation is stripped; an inline `>` stays text.
- Exempting by the phrase's grammatical role (subject noun phrase + predicate): it would silence the sentence the operator placed on the fire side.

## Related

#124 (umbrella, parent) · #918 / PR #985 (executed-report exemption) · #958 (declined boundary) · #78 (Vega — the operator-dialogue carve axis) · neo#14437 (use/mention carve) · neo#16411 (citation bridge) · #136 (parent epic of #124).

Origin Session ID: 5f1fc45a-d345-4bc2-92a2-780c0822a6dd (rotated mid-session to 72777b62-d67b-42f2-8ac5-e09a717c0def)
Retrieval Hint: "deference matcher your call temporal negation confession blockquote" · "stop hook false positive adjacency anchor"

Live latest-open + closed sweep: latest 20 open (created-desc) at 2026-10-31 00:07Z — no equivalent; closed search (deference stop hook false positive your call) → #918 COMPLETED (executed reports; kept as the precedent shape), #958 NOT_PLANNED (boundary respected, Out of Scope above), #137 NOT_PLANNED. A2A in-flight sweep: the latest 30 `[lane-claim]` rows at 00:07Z — only my own claim on #124 (00:00Z). Memory Core rationale sweep: `query_raw_memories` on the problem nouns returned no relevant row (the thread is the record). Own-assignment sweep: 21 open, none overlapping. Structure map: `ai/scripts/lifecycle` owns the matcher; no new file.


## Timeline

- 2026-10-31T00:09:25Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-10-31T00:09:25Z @neo-fable-mnemosyne assigned to @neo-fable-mnemosyne
- 2026-10-31T00:09:25Z @neo-fable-mnemosyne added the `ai` label
- 2026-10-31T00:09:25Z @neo-fable-mnemosyne added the `testing` label
- 2026-10-31T00:09:25Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-10-31T00:11:05Z @neo-fable-mnemosyne added parent issue #124
- 2026-10-31T00:11:21Z @neo-fable-mnemosyne cross-referenced by #124
- 2026-10-31T00:17:24Z @neo-fable-mnemosyne cross-referenced by PR #1388
- 2026-10-31T00:21:36Z @neo-fable-mnemosyne referenced in commit `d248aa7` - "test(hooks): the carve tables name their sources without a ticket id (#1387)"
- 2026-10-31T00:55:54Z @tobiu referenced in commit `941b312` - "fix(hooks): carve a cited, negated or confessed "your call" from the deference matcher (#1387) (#1388)

* fix(hooks): carve a cited, negated or confessed "your call" from the deference matcher (#1387)

* test(hooks): the carve tables name their sources without a ticket id (#1387)"
- 2026-10-31T00:55:56Z @tobiu closed this issue

