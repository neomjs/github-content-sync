---
id: 1420
title: 'The deference matcher fires on a relayed decision: "relayed your call"'
state: CLOSED
labels:
  - bug
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-01T22:57:03Z'
updatedAt: '2026-11-02T15:11:33Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1420'
author: neo-opus-grace
commentsCount: 1
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
closedAt: '2026-11-02T15:11:33Z'
---
# The deference matcher fires on a relayed decision: "relayed your call"

## Context

Filed under the Stop hook's own friction→gold clause ("if it fired wrong — a false positive … open a ticket to sharpen it"). On 2026-11-01 at about 23:55 CET, the Claude Code Stop hook in a `neomjs/neo` session fired `deference phrase "your call" at turn-terminal` on this closing-message sentence:

> **Euclid also relayed** your call to pause the film.

The sentence reports that a peer had relayed a decision the operator already made. Nothing is handed back to the operator. This is the second occurrence of #918's class: an executed-report verb that `CITATION_ANCHOR`'s closed list does not carry.

- Live latest-open + closed sweep: the latest 20 open (created-desc) at 23:00Z; the closed search "deference citation anchor verb your call" → #918 and #1387, with their Problem, Fix, Out of Scope and Avoided Traps read. #124, the specimen umbrella, is past the comment cap, so this is a leaf, as #918 and #1387 were. No equivalent found.
- A2A in-flight claim sweep (last 60 minutes): no claim on the matcher.
- Memory Core rationale sweep ("deference stop hook fires on a reported decision relayed by a peer"): no prior specimen with this verb.
- Own-assignment sweep: #684 and #54, unrelated.
- Structure map: N/A. This is one data change in an existing module, with no placement change.

## The Problem

`isAttributiveCitationContext` exempts `your call` only when an anchor sits immediately before the phrase. The anchors are `per`, `as you said/directed/called`, a duration before `after|since` (#1387), and the executed-report verbs `recorded|followed|implemented|applied|took|went with` (#918). On `dev`, `ai/scripts/lifecycle/deferencePhraseMatch.mjs:257` (`CITATION_ANCHOR`) has no `relayed`. "relayed your call" reports the same thing "recorded your call" does, a decision already made and passed on, yet it fires. Each firing costs the agent one forced continuation.

## The Architectural Reality

- The matcher is `ai/scripts/lifecycle/deferencePhraseMatch.mjs` (`matchDeferencePhrase`, `isAttributiveCitationContext`, `CITATION_ANCHOR`, `isCitationBridge`). Its one consumer is `ai/scripts/lifecycle/stopHookDecision.mjs` (`decideDeferenceStopHookAction`), under the Claude and Codex Stop hooks.
- The spec is `test/playwright/unit/ai/scripts/lifecycle/deferencePhraseMatch.spec.mjs`, with its `reports`, `handbacks` and `existingExemptions` tables.
- The declined boundaries hold:
  - no exemption by grammatical role (#1387's avoided trap: "your call to revert … was right" stays on the fire side);
  - no human-owned-domain exemption (#958).

## The Fix

Add `relayed` to the executed-report verbs in `CITATION_ANCHOR`; the adjacency rule is unchanged. Add one spec row on each side, naming this ticket.

## Acceptance Criteria

- [ ] AC-1: silent on "Euclid also relayed your call to pause the film."
- [ ] AC-2: still fires on "Relayed. Your call?", on "I relayed it, your call.", and on every existing control in the spec.
- [ ] AC-3: `npm run test-unit -- test/playwright/unit/ai/scripts/lifecycle/deferencePhraseMatch.spec.mjs` and `stopHookDecision.spec.mjs` are green.

## Out of Scope

- Reporting verbs nobody has observed yet (`conveyed`, `forwarded`, `passed on`): they're added when a specimen appears, as #918 and #1387 did.
- Grammatical-role exemptions (#1387) and the human-owned-domain exemption (#958).

## Avoided Traps

- A window-wide `relay` anchor: "Should I relay it? Your call." is a handback. Only the adjacent past-tense verb is a report.

## Related

#124 (the umbrella) · #918 (the verb-list precedent) · #1387 (the last batch) · #136 (mechanical enforcement epic)

unowned-rationale: Ready for a contributor; maintainers provide planning and review.

Origin Session ID: f03e1a63-416e-4c52-bc19-35dd647c2b63


## Timeline

- 2026-11-01T22:57:04Z @neo-opus-grace added the `bug` label
- 2026-11-01T22:57:04Z @neo-opus-grace added the `good first issue` label
- 2026-11-01T22:57:04Z @neo-opus-grace added the `ai` label
### @ShahadatTest - 2026-11-02T00:01:12Z

I'll add `relayed` to the deference citation-anchor verb list so "relayed your call" is treated as an executed-report citation, matching `recorded|followed|...`. AI-assisted.

- 2026-11-02T00:03:16Z @ShahadatTest cross-referenced by PR #1424
- 2026-11-02T02:25:56Z @neo-fable-mnemosyne cross-referenced by #1425
- 2026-11-02T15:11:33Z @tobiu referenced in commit `59c5a55` - "fix(lifecycle): treat "relayed your call" as an executed-report citation (#1420) (#1424)

Add relayed to CITATION_ANCHOR's executed-report verb list so peer relay
sentences do not trip the Stop-hook deference matcher. Spec covers the
specimen and the adjacent handback controls.

AI-assisted.

Co-authored-by: Cursor Agent <cursoragent@cursor.com>"
- 2026-11-02T15:11:34Z @tobiu closed this issue

