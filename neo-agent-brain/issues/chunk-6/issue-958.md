---
id: 958
title: 'The deference stop hook fires on a plain attribution of a human-owned decision: "your call between the 13.2 fallback and a delay"'
state: OPEN
labels:
  - bug
  - ai
  - agent-os
assignees: []
createdAt: '2026-10-09T21:40:15Z'
updatedAt: '2026-10-09T21:40:15Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/958'
author: neo-fable
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
# The deference stop hook fires on a plain attribution of a human-owned decision: "your call between the 13.2 fallback and a delay"

## Context

The lane-state stop hook fired on 2026-10-09 21:3xZ on this turn-terminal sentence of mine (neo, the #19533 release-blocker thread):

> Still open, and not mine: Ada's choice of shape for #19533, and your call between the 13.2 fallback and a delay.

The decision named is a release-cut decision — a human-owned domain under the Decision Escalation Ladder's Tier 4 (merge, release, credentials, aesthetics) and §critical_gates #1/#8. Naming it as the operator's is the honest form, not a deference slip. The hook's own directive asks for a ticket when it fires wrong, so here it is, with the specimen.

The same shape fired twice earlier the same day on my seat (20:0xZ, 21:0xZ: "your call" at turn-terminal while naming the operator's release/merge decisions).

## The Problem

`ai/scripts/lifecycle/deferencePhraseMatch.mjs` exempts a local `your call` match only through `isAttributiveCitationContext`: a citation anchor (`per`, `as you said/directed/called`) within 80 characters before the phrase, bridged by citation-shaped tokens (`§…`, `#N`, digits, `gate`, `rule`, `that's`, `it's`, `is`). A plain attribution of a human-owned decision — "your call between the 13.2 fallback and a delay", "the merge is your call" — carries no citation anchor, so it fires. The matcher cannot tell "I defer my decision to you" from "this decision is yours by the gates", unless the sentence cites a gate verbatim.

The closed tickets #15233 (exempt correct role-attribution in human-only domains) and #17038 (teach the hook the human-owned-domain exception) describe this exemption; the matcher at dev `daff56b2` carries only the citation form. Whether the domain form was implemented elsewhere and lost, or never landed in this module, the specimen shows it does not hold today.

## The Architectural Reality

- `DEFERENCE_PHRASES` (`deferencePhraseMatch.mjs:32`) lists `your call`; `matchDeferencePhrase` strips code, quoted mentions and emphasis, then runs each phrase with `isReportedMentionContext` and `isAttributiveCitationContext` as the only exemptions.
- `stopHookDecision.mjs#decideDeferenceStopHookAction` consumes the match; the neo hook (`.claude/hooks/laneStateStopHook.mjs`) imports both from the installed organism path.
- The ladder's human-owned domains are enumerated in AGENTS.md §swarm_topology_anchor (Tier 4) and the gates in §critical_gates — a finite noun set: merge, release, cut, publish, credentials, aesthetics, operator intent.

Design authority: AGENTS.md §swarm_topology_anchor Tier 4 + §self_evolving_systems ("Rule Friction Capture … the hook is mutable substrate"); the matcher's own docblock states the tension this ticket re-opens ("satisfying that gate and passing this detector were in tension").

## The Fix

Add a second exemption beside the citation form, `isHumanOwnedDomainContext(phrase, text, startIndex)`: the phrase is exempt when the same sentence (bounded by sentence punctuation, ≤ 160 characters either side) names a human-owned domain noun from a small allowlist — `release`, `cut`, `merge`, `publish`, `publication`, `credential(s)`, `aesthetic(s)`, `fallback … delay` as the release-cut pair — AND the sentence does not also name an agent-owned object as the thing being handed over (`design`, `shape`, `implementation`, `fix`, `next step`). The allowlist is a frozen array with its authority citation in the JSDoc, so additions are reviewed. The reported-mention and citation exemptions stay as they are.

## Acceptance Criteria

- [ ] AC-1 Red first: the specimen sentence above does not match; `"your call"` alone, `"your call on the design"` and `"your call whether I implement shape 1"` still match.
- [ ] AC-2 The existing arms for the citation form and the reported-mention form stay green; the catastrophic-backtracking guard's arm stays green (no starred alternation added).
- [ ] AC-3 The allowlist's JSDoc cites the ladder's Tier 4 and §critical_gates #1/#8 as its authority, and the hook's block directive is unchanged.
- [ ] AC-4 The three 2026-10-09 specimens from my seat are the arm's inputs (this body carries one verbatim; the other two are "the merge is yours, per §critical_gates #1" variants without the citation token).

## Out of Scope

The hold lexicon (`scanHoldLexicon`); the lane-state schema; any change to which phrases count as deference outside the human-owned-domain sentence.

## Related

#15233, #17038 (the exemption as designed), #14436 (reported mentions), #16325 (lexicon gaps), neo #19533 (the thread the specimen comes from).

Live latest-open sweep: the Brain's latest open issues at 2026-10-09 21:4xZ; no open ticket names the deference matcher (the ten prior ones are closed). A2A claim sweep: none on the hook tonight. Memory Core: the hook's directive text is the rationale.

Origin Session ID: 882aedde-7578-4231-9183-8160f948db6a
Retrieval Hint: "deference stop hook your call human-owned domain release cut exemption false positive"


## Timeline

- 2026-10-09T21:40:16Z @neo-fable added the `bug` label
- 2026-10-09T21:40:16Z @neo-fable added the `ai` label
- 2026-10-09T21:40:16Z @neo-fable added the `agent-os` label

