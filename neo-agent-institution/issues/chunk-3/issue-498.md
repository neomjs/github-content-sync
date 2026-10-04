---
id: 498
title: 'Row 2''s walkthrough, fixture half: six states read on every cockpit surface'
state: CLOSED
labels:
  - enhancement
  - agent-os
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-10-03T10:56:49Z'
updatedAt: '2026-10-04T12:14:40Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/498'
author: neo-opus-grace
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
blocking:
  - '[ ] 479 Row 2''s installed walkthrough: the five states provoked on one candidate, one receipt per state'
closedAt: '2026-10-04T12:14:40Z'
milestone: FM v1
---
# Row 2's walkthrough, fixture half: six states read on every cockpit surface

## Context

#479 is row 2's installed walkthrough, the operator-slot packet for FM v1 ROADMAP row 2. Its fixture arm was open to any seat that ran the packaged smoke; I took it and built it as PR #494. Row 2's steward (Clio) chose to keep #479 open as the slot's packet, the way row 3 splits #485 (the slot) from #486 (the fixture read). So the fixture half gets its own leaf, and #479 is blocked by it.

## The Problem

The census (`learn/CockpitStateCensus.md`, #478) says what each cockpit surface claims in each state. Nothing has provoked those states and read the surfaces against it. Its 10 candidate gaps are unconfirmed, and an operator slot spent finding what a fixture could show costs the scarcest seat's time.

## The Architectural Reality

- `FleetCockpitLivenessNL.spec.mjs` already drives the mounted cockpit against a real Fleet server, with the real activity producer and transport kill and restart, and proves the state transitions.
- The census surfaces on the cockpit page are the spine banner, the instance switcher, the roster and the activity feed. Home, the query-time panes, the plane probe and the tray are elsewhere or shell-only.
- Brain health reaches the cockpit only from the shell's lifecycle owner (`Neo.Main.brainHealth`). In a browser, the cockpit's `applyBrainHealth` handle carries the degraded state.

## The Fix

As built in PR #494:
- `CockpitStateWalkthroughNL.spec.mjs` provokes cold, unreachable, live, stale, one source failing and degraded on one cockpit. It asserts every surface on the page against its census cell and attaches one receipt per state, with the Institution and Brain revisions.
- `learn/CockpitStateWalkthrough.md` is the script: each provocation marked fixture-runnable or slot-only, and what only the slot reads.
- The census's switcher cells for stale and one source failing are corrected.

## Acceptance Criteria

- [ ] AC-1: The walkthrough page names the six provocations, each fixture-runnable or slot-only, and what only the slot reads.
- [ ] AC-2: The fixture arm runs green on a checkout and its Brain pin. Each state's words equal the census cell on every surface of the cockpit page (e2e, NL).
- [ ] AC-3: One receipt per state is on #479, each naming both revisions and the words observed.

## Out of Scope

The operator's slot, the remaining surfaces and the ROADMAP cell (#479); fixing any confirmed gap (its own leaf under #477).

## Related

Parent: #477. Blocks: #479. The census: #478. Sibling shape: #485 and #486.

Live latest-open sweep: the latest 20 open Institution issues, read at 2026-10-03T10:56:31Z. No equivalent: #479 is the slot this splits from. A2A: the steward's shape call (option 2). Own-assignment: #479 (the slot), #486, #490, #414 and #11; none duplicates this.

Origin Session ID: 9eba4853-ea86-428a-85f9-e9060002ca22
Retrieval Hint: "row 2 walkthrough fixture half census executable six states cockpit surfaces"

🖖 Grace (Claude Opus 5.5, Claude Code)

## Timeline

- 2026-10-03T10:56:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-03T10:56:50Z @neo-opus-grace added the `enhancement` label
- 2026-10-03T10:56:50Z @neo-opus-grace added the `agent-os` label
- 2026-10-03T10:56:50Z @neo-opus-grace added the `ai` label
- 2026-10-03T10:56:50Z @neo-opus-grace added the `testing` label
- 2026-10-03T10:56:56Z @neo-opus-grace added parent issue #477
- 2026-10-03T10:56:58Z @neo-opus-grace marked this issue as blocking #479
- 2026-10-03T10:56:58Z @neo-opus-grace added this to the **FM v1** milestone
- 2026-10-03T10:57:32Z @neo-opus-grace cross-referenced by PR #494
- 2026-10-03T11:07:16Z @neo-opus-vega cross-referenced by PR #482
- 2026-10-03T17:40:37Z @neo-fable-clio cross-referenced by #477
- 2026-10-04T11:03:05Z @neo-opus-grace cross-referenced by #15000
- 2026-10-04T11:21:31Z @neo-opus-grace referenced in commit `b523926` - "test(e2e): the walkthrough disposes its servers and activity source on a rejected step (#498)

Every Fleet server the walkthrough starts is tracked and closed in a finally, and the bridge's
activity source is put back to the one the run found, so a rejected receipt leaks neither a
listening port nor a missing-corpus source into the next spec. A second test rejects inside the
same helper and checks both. The spec and the walkthrough page now say the census is a literal
copied by hand from the census page, which the spec never reads."
- 2026-10-04T11:56:03Z @neo-gpt-emmy cross-referenced by #12
- 2026-10-04T12:14:40Z @tobiu referenced in commit `77827bd` - "test(e2e): row 2's walkthrough reads every cockpit surface in six provoked states (#498) (#494)

* test(e2e): row 2's walkthrough reads every cockpit surface in six provoked states against a real Fleet server (#479)

The fixture half of the walkthrough: cold, unreachable, live, stale, one source failing and degraded, provoked in turn on one mounted cockpit. Each state's words must equal its census cell and are attached as that state's receipt, beside the Institution and Brain revisions the run read. The real-server helpers move from the liveness spec into the shared Fleet harness.

* docs(learn): the state walkthrough's script, and the switcher's stale and partial cells corrected (#479)

The page names each provocation as fixture-runnable or slot-only, and what only the slot reads. The census's instance switcher reads degraded in the stale and one-source-failing states: its word follows the spine banner's kind.

* test(e2e): the walkthrough disposes its servers and activity source on a rejected step (#498)

Every Fleet server the walkthrough starts is tracked and closed in a finally, and the bridge's
activity source is put back to the one the run found, so a rejected receipt leaks neither a
listening port nor a missing-corpus source into the next spec. A second test rejects inside the
same helper and checks both. The spec and the walkthrough page now say the census is a literal
copied by hand from the census page, which the spec never reads."
- 2026-10-04T12:14:40Z @tobiu closed this issue

