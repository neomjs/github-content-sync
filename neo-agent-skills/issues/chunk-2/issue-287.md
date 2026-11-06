---
id: 287
title: The e2e protocol never says to wait for a lazily loaded plugin
state: CLOSED
labels:
  - documentation
  - enhancement
  - ai
  - testing
assignees:
  - neo-opus-grace
createdAt: '2026-11-06T00:57:26Z'
updatedAt: '2026-11-06T11:46:07Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/287'
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
closedAt: '2026-11-06T11:46:07Z'
---
# The e2e protocol never says to wait for a lazily loaded plugin

## Context

Two engine flakes surfaced on 2026-11-05/06 with one cause: the spec acted on first paint, before a plugin its component imports lazily had installed.
- **`grid/IssuesShowcase.spec.mjs`:** its one-shot accessibility read ran before `grid.header.Button#afterSetColumnMenu` had imported `plugin/Menu.mjs`, so no filter trigger existed yet. Three reds; neomjs/neo#20407, PR neomjs/neo#20408.
- **`rendering/CalendarMonthEventDrag.spec.mjs`:** its `page.mouse` drag started before `calendar.view.month.Component#afterSetEnableDrag` had imported `plugin/DragDrop.mjs`, so the drag never armed. Red in 3 of 21 runs; fixed in PR neomjs/neo#20404.

## The Problem

`whitebox-e2e-protocol.md` §3 tells an author to interact through Playwright and then assert engine truth. It says nothing about what must exist before the first interaction. A `toBeVisible()` on the component passes on first paint, and Neo installs a lazily imported plugin after that.

Design authority: §3's interact-then-assert pattern, read with Playwright's actionability rule that a test acts only on state it has observed. First paint does not establish that a lazily installed plugin exists. This ticket states that test-correctness boundary and changes no product behavior.

## The Fix

Add one short rule to §3, in the payload that every e2e authoring load reads. Before the first gesture or one-shot read, wait on the observable effect of what the test acts on: the lazily installed plugin's own DOM, or its registered listener. First paint is not that signal, and neither is a sleep. Bump the package version.

Slot disposition: `keep`, `DISCIPLINE-ONLY`. Retirement trigger: retire the rule once the e2e fixture offers a settle wait that covers a component's pending lazy imports.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Edge | Docs | Evidence |
|---|---|---|---|---|---|
| `whitebox-e2e-protocol.md` §3 | #287, the two lazy-plugin sources above, and the neomjs/neo#20408 witness | Before a gesture or one-shot read that depends on a lazily installed plugin, wait for that plugin's own DOM node or listener | First paint or a fixed sleep does not prove the plugin is installed | The §3 sentence | Corpus and reference lint; the neomjs/neo#20408 reproducer (2.5 s hold: red 3/3 → green 3/3); the readiness poll in neomjs/neo PR #20404 |

## Acceptance Criteria

- [ ] `whitebox-e2e-protocol.md` §3 states the readiness rule, anchored on a source spec. The corpus growth cap fits one anchor; the other spec is named here.
- [ ] The PR bumps the package version and names the retirement trigger.

## Out of Scope

- A fixture-level settle primitive. That is the retirement path, and its own ticket if wanted.
- The two spec fixes (PR neomjs/neo#20408, PR neomjs/neo#20404).

## Related

neomjs/neo#20407 · neomjs/neo#20374

Ownership: self-assigned; not a `good first issue` because the edit is in progress.
Sweeps: latest 20 open plus closed search (whitebox e2e readiness, lazy plugin, flake), recent A2A lane claims, Memory Core and my own assignments, at 2026-11-06T00:56Z. No equivalent ticket, claim or prior attempt.
Origin Session ID: 57544c35-7c06-4a02-b04e-860846ee64a8
Retrieval Hint: "e2e test races lazily imported plugin first paint readiness whitebox protocol"


## Timeline

- 2026-11-06T00:57:26Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-11-06T00:57:27Z @neo-opus-grace added the `documentation` label
- 2026-11-06T00:57:28Z @neo-opus-grace added the `enhancement` label
- 2026-11-06T00:57:28Z @neo-opus-grace added the `ai` label
- 2026-11-06T00:57:28Z @neo-opus-grace added the `testing` label
- 2026-11-06T00:58:31Z @github-actions added the `needs-design-authority` label
- 2026-11-06T00:59:45Z @neo-opus-grace cross-referenced by PR #288
- 2026-11-06T01:38:22Z @neo-opus-grace removed the `needs-design-authority` label
- 2026-11-06T02:10:05Z @neo-opus-ada cross-referenced by #20420
- 2026-11-06T10:48:18Z @neo-opus-grace cross-referenced by #292
- 2026-11-06T11:46:07Z @tobiu referenced in commit `a49455d` - "docs(whitebox-e2e): wait for what the test acts on, not first paint (#287) (#288)

A lazily imported plugin installs after the component's first render, so a gesture or one-shot read can beat it. Two engine flakes had that cause. Bumps the package to 0.1.91."
- 2026-11-06T11:46:08Z @tobiu closed this issue

