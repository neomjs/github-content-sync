---
id: 59
title: 'Cockpit bar IA: status-word pills, state block, declared collapse order'
state: CLOSED
labels: []
assignees: []
createdAt: '2026-08-29T23:34:21Z'
updatedAt: '2026-08-30T00:00:16Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/59'
author: neo-fable-clio
commentsCount: 0
parentIssue: 23
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-08-30T00:00:16Z'
---
# Cockpit bar IA: status-word pills, state block, declared collapse order

## Problem Scope

Implementation cut A of the #23 IA sketch (`apps/agentos/design/institution-header-detail-ia.html`, merged via PR #57 after the operator's AC-1 read): the cockpit bar still rendered the pre-sketch shape — a five-fact prose banner BETWEEN action buttons, a wake chip clipping mid-word, no declared item order or collapse behavior.

## Solution Shape

Consume the sketch's header sections verbatim: declared item order (views → spacer → STATE block → actions), status-word pills with the full honesty sentences on title/aria (derivation modules return `{hidden, kind, text, title, ariaLabel}`; every retention/partition/ranking doctrine unchanged), and the declared collapse order via container queries on the existing `fm-cockpit` context.

## Acceptance Criteria

- [ ] The cockpit bar renders the four item classes in the declared order; state never sits between action buttons (one right-aligned block before the actions).
- [ ] Chrome labels are status words; no label carries an endpoint, shell command, or sentence inline — full truth on title + aria (screen-reader reachable).
- [ ] Declared collapse order: state pills stack vertically wide, run in one row ≤ ~1180, drop to marks-with-titles ≤ ~730; action labels drop to glyphs; view labels never drop; no mid-word clipping at any step (witnessed).
- [ ] The wake chip never repeats its axis word ("wake: wake …" is unrepresentable).
- [ ] Full unit + component batteries green; visual goldens deliberately re-captured; baseline stamp index-true.

## Out of Scope

The agent-detail rail (its own cut under #23) · the typed connection-state pill vocabulary (#18/#15 feed the pill text when they land) · §04 chip-family unification of the two state chips (#24 leaf).

## Related

Parent: #23 (AC-2 of its checklist). Sketch: PR #57. Consumer: the open cockpit-bar PR.

Authored by Clio (Fable 5, Claude Code). Origin Session ID: 729e92a3-fa8a-4c4f-bc72-6c0ad1a5e604

Retrieval Hint: `query_raw_memories("cockpit bar status word pills state block collapse order")`


## Timeline

- 2026-08-29T23:35:03Z @neo-fable-clio cross-referenced by PR #58
- 2026-08-29T23:35:45Z @neo-fable-clio cross-referenced by #23
- 2026-08-30T00:00:16Z @tobiu closed this issue

