---
id: 788
title: ADR 0041 §3 names how a host-file effect settles
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - architecture
  - agent-os
assignees:
  - neo-fable
createdAt: '2026-10-03T06:59:36Z'
updatedAt: '2026-10-03T09:10:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/788'
author: neo-fable
commentsCount: 0
parentIssue: 351
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 786 An interrupted file effect deadlocks the first run on a cold host'
closedAt: '2026-10-03T08:27:29Z'
---
# ADR 0041 §3 names how a host-file effect settles

## Context

The Decision Record half of #786. ADR 0005 §6.5 puts an ADR revision in its own PR ahead of the implementation that needs it, so the sentence lands here and #786's PR cites it as merged authority.

## The Problem

ADR 0041 §3's witness ends: *no step turns green until a fresh observation from the matching plane — id and root — arrives.* Read for every effect, that sentence cannot be met on a cold host: `write-secrets` and `write-env` exist to bring the plane up, so an interrupted one waits for a plane that only the effects behind it can start (#786 carries the measurement). §3 states the rule for the plane's own effect and is silent on an effect that precedes the plane; the orchestration implemented the silence as "the plane, for all three".

## The Architectural Reality

- `learn/agentos/decisions/0041-bootstrap-record-verified-plane-handoff.md` §2 item 6: an ambiguous effect resolves to `reconcile-required` and is settled only by a fresh matching observation, never by replay. Unchanged by this leaf.
- §3, first paragraph: the witness every implementing leaf inherits. §6 makes it required test evidence.
- `ai/services/fleet/firstRunRecipe.mjs#evaluateEffect`: an effect with no receipt whose result is present already reads `ok` from presence alone, so for a host file the plane gate adds no safety the recipe keeps on any other path. The identity gate for those files is the `served-plane` row downstream.

## The Fix

One sentence appended to §3's first paragraph:

> One kind of effect is matched without the plane: a host-file effect ordered before the plane's own exists to bring that plane up, so it settles by the file's own fresh matching observation — present, unproblematic, and the content its receipt expected where it recorded one; the plane's effect and every step after it settle only by the plane's.

The Status row names the amendment with its date and its leaf.

## Decision Record impact

amends ADR 0041 §3 by one sentence. The record's author assented by A2A on 2026-10-03 (`cba07660`) and reads it as a clarification of §2.6's "matching observation".

## Acceptance Criteria

- [x] AC-1 §3 carries the sentence; §2 item 6 is unchanged.
- [x] AC-2 The Status row names the amendment, its date and its leaf (the row is written before a PR number exists).
- [x] AC-3 No other sentence in `learn/` restates the old reading (grep receipt in the PR). The existing `ai/services/fleet/setupOrchestration.mjs#settlePending` summary is updated by the separate implementation leaf #786 / PR #790, after this ADR amendment, as already declared in PR #789's AC evidence.

## Out of Scope

The implementation and its witness arms (#786).

## Related

#786 (blocked by this) · neomjs/neo-agent-institution#351 (parent) · #678 (the leaf that published the record) · ADR 0005 §6.5

Live latest-open sweep: the latest 20 open issues at 2026-10-03T06:59Z (newest #787) and an `ADR 0041` title search over open and closed issues: no equivalent (#678 is the publishing leaf, closed).
A2A in-flight sweep at 06:59Z, the latest 30 messages of every read-state: no claim on ADR 0041.
MC sweep: shared with #786 ("first-run setup stuck: an interrupted effect stays reconcile-required and nothing after it runs on a host where no plane is serving yet"), 5 results, no prior decision found.
Own-assignment sweep: 8 open, #786 the only one on this surface.

Origin Session ID: 25618ee4-58d2-46dd-ae26-9dcf2854b14a
Retrieval Hint: "ADR 0041 §3 host-file effect settles by its own observation amendment"

🪢 Mnemosyne (Claude Fable 5.1 · Claude Code) · session 25618ee4-58d2-46dd-ae26-9dcf2854b14a



## Timeline

- 2026-10-03T06:59:36Z @neo-fable assigned to @neo-fable
- 2026-10-03T06:59:37Z @neo-fable added the `documentation` label
- 2026-10-03T06:59:37Z @neo-fable added the `enhancement` label
- 2026-10-03T06:59:37Z @neo-fable added the `ai` label
- 2026-10-03T06:59:38Z @neo-fable added the `architecture` label
- 2026-10-03T06:59:38Z @neo-fable added the `agent-os` label
- 2026-10-03T06:59:59Z @neo-fable cross-referenced by #786
- 2026-10-03T07:00:00Z @neo-fable added parent issue #351
- 2026-10-03T07:00:01Z @neo-fable marked this issue as blocking #786
- 2026-10-03T07:02:40Z @neo-fable cross-referenced by PR #789
- 2026-10-03T07:05:49Z @neo-fable cross-referenced by PR #790
- 2026-10-03T07:08:41Z @neo-fable cross-referenced by #792
- 2026-10-03T08:27:29Z @tobiu referenced in commit `a96d857` - "docs(adr): ADR 0041 §3 names how a host-file effect settles (#788) (#789)

A host-file effect ordered before the plane's own exists to bring that plane up, so it settles by the file's own fresh matching observation; the plane's effect and every step after it settle only by the plane's. One sentence in §3, the amendment named in the Status row; §2 item 6 is unchanged."
- 2026-10-03T08:27:30Z @tobiu closed this issue
- 2026-10-03T08:41:48Z @neo-fable-clio cross-referenced by #802
- 2026-10-03T08:42:36Z @neo-fable-clio cross-referenced by PR #803

