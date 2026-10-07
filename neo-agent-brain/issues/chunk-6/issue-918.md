---
id: 918
title: 'The deference mirror fires on a reported decision: "recorded your call"'
state: OPEN
labels:
  - bug
  - ai
assignees: []
createdAt: '2026-10-07T12:51:47Z'
updatedAt: '2026-10-07T12:51:47Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/918'
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
# The deference mirror fires on a reported decision: "recorded your call"

## Context

2026-10-07, neomjs/neo session `9aa8aa9b`: the deference stop hook blocked a turn whose final message opened with "I recorded your call as neomjs/neo#19450: …". The operator had made the decision in the previous prompt; the sentence reports it being carried out. The hook matched `your call` (`ai/scripts/lifecycle/deferencePhraseMatch.mjs:32`) and asked the agent to stop deferring. Nothing was being deferred.

## The Problem

`deferencePhraseMatch.mjs` already exempts two non-deferential uses: a reported mention (`isReportedMentionContext`: "the phrase …", "quoted …") and a citation of a prior decision (`CITATION_ANCHOR`: `per …`, `as you said/directed/called`). A third common form has no exemption: a past-tense verb **reporting that an operator decision was carried out**, with "your call" as its object. Examples: "I recorded your call", "followed your call", "went with your call". Each one fires the mirror on a turn that did the opposite of deferring.

## The Fix

Extend the citation exemption with an allowlist of executed-decision verbs immediately before the phrase: `(?:recorded|followed|implemented|applied|took|went\s+with)\s+your\s+call`. Keep it an allowlist and adjacency-anchored, as `CITATION_ANCHOR` is, so "your call?" anywhere later in the same message still fires (the existing rightmost-anchor rule).

## Acceptance Criteria

- [ ] "I recorded your call as #N: …" and "went with your call" do not fire (unit).
- [ ] "recorded it — your call?" and a message ending "**Your call:** A or B?" still fire (unit).
- [ ] Existing `deferencePhraseMatch` arms pass unchanged.

## Out of Scope

Any other phrase in the lexicon; the hook's lane-state behaviour.

## Related

The citation-anchor rationale in `deferencePhraseMatch.mjs` (`CITATION_ANCHOR`, `isCitationBridge`).

Live sweep: Brain issues "deference phrase false positive stop hook" and "deference mirror referential": none.

unowned-rationale: a one-regex fix; a builder self-selects, and my Claude budget is tight until Thursday.

Origin Session ID: 9aa8aa9b-2502-458b-976b-eec8a223218e


## Timeline

- 2026-10-07T12:51:48Z @neo-opus-grace added the `bug` label
- 2026-10-07T12:51:49Z @neo-opus-grace added the `ai` label

