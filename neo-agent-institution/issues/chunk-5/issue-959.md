---
id: 959
title: Home counts the asks that expired unanswered
state: OPEN
labels:
  - enhancement
  - ai
  - design
assignees:
  - neo-fable-clio
createdAt: '2026-11-13T20:29:16Z'
updatedAt: '2026-11-13T22:25:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/959'
author: neo-fable-clio
commentsCount: 1
parentIssue: 414
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
# Home counts the asks that expired unanswered

## Context

neomjs/neo-agent-brain#1677 (pull request neomjs/neo-agent-brain#1684) lets an ask to a human expire: at its stated deadline, or seven days after it was sent. The operator said yes to that on 2026-11-13 ([the decision receipt](https://github.com/neomjs/neo-agent-brain/issues/1677#issuecomment-6861670460)). By #951's two readings the first sweep on our plane ends about 141 of the 146 asks that Home counts today.

The design seat's answer to "an ask leaves his view without his act" is [on that ticket](https://github.com/neomjs/neo-agent-brain/issues/1677#issuecomment-6849306319): the ask does not leave, it moves from waiting to expired unanswered, a second and quieter number on Home that opens the list of exactly those. The sweep's runner, neomjs/neo-agent-brain#1679, goes live only once this count is installed.

This is the leaf #951 announced for the day #1677's pull request opens.

## The Problem

- Home's operator line has two axes, questions and merges (`apps/agentos/view/home/Container.mjs`, `operatorLine`). After the first sweep it reads `5 questions wait for you`, and nothing on Home says that 141 asks ended without a word.
- The Mailbox can show one expired ask (`view/fleet/mailbox/DetailContainer.mjs`, `getExpiredText`: `expired; planned fallback: …`), and no list leads to them. `for you` reads open Tasks only (`util/OperatorInbox.mjs`, through `fleetOwnQuestions`), and in `all mail` an ask from July sits many pages down.
- The producer has no such number. `fleetOpenWork`'s `questions` block is `{state, count, reason}` over `InputRequired`, `Submitted` and `Working` (Brain, `ai/services/fleet/FleetControlBridge.mjs`, `readOwnQuestions`), and `MailboxService.listMessages` filters by `taskStates` and by no time.

## The Architectural Reality

- `Design authority: n/a — adds a count and a list; the line's two counts keep their rule (#551).`
- **The window is the producer's own.** The sweep sets `task.state = 'Expired'` and, with the producer's leaf, stamps `task.expiredAt` in the same statement; the read windows on it over `mailbox.humanAskTtlMs`. The app computes no window: it reads `since`.
- **The wire** (the producer's leaf neomjs/neo-agent-brain#1689, filed by the read's owner as #951 prescribes; pull request neomjs/neo-agent-brain#1691, names read from its diff at `b7b01c4660` on 2026-11-13): `fleetOpenWork()` answers `questions: {state, count, reason, expired: {state, count, since, reason}}`. `fleetOwnQuestions({expired: true, limit, offset})` answers `{state: 'ok', reason: null, count, rows, page, since}`: the mirror's body-free rows, newest expiry first, `since` the window's start as an ISO instant. A failed read answers `state: 'unavailable'` with its `reason`, `count: null`, no rows and `since: null`. No new wire method.
- **A plane without that leaf.** `util/OpenWorkRead.questions` already answers a wire without a `questions` block as `unsupported`. One level down, a block without `expired` adds nothing. The bridge on such a plane ignores `expired: true` and answers the open list (`FleetControlBridge.fleetOwnQuestions` reads `limit` and `offset` only), so a list answer without a string `since` is not the expired list.
- **Home's links.** A count is a segment `{link, text}`; a click on it fires `questionsOpen` or `mergeQueueOpen` (`onFieldClick`), and `ViewportController.onHomeQuestionsOpen` routes to `/fleet` and calls the cockpit's `openOperatorQuestions()`.
- **The Mailbox's switch.** `OperatorContainer` carries the chips `all mail` and `for you · 3 open`; `mailbox/Container` has `view: 'open'` beside the mirror, with its own states.

## The Look

Decided here, before the build.

| State | Home's line |
|---|---|
| Asks wait, asks expired | `5 questions · 2 merges wait for you · 141 expired unanswered` |
| Nothing waits, asks expired | `nothing waits for you · 3 expired unanswered` |
| No ask expired in the window | as today, no third run |
| The count is `null` or absent | as today, no third run and no zero |

- The third run comes last, so the line still opens with what waits. It is a link in the colour of the line's plain words, underlined on hover like the counts. The two waiting counts keep their accent; the expired count never takes it and never takes a warning colour.
- Its title says what the number is: `asks whose deadline passed without an answer since <date>`, from the window's start.
- In the Mailbox a third chip, `expired · 141`, stands beside `for you · 5 open` while the count is above zero. It switches the list to those asks, newest expiry first. A row opens the detail, which already says what its sender planned.
- The list names its own states in the pane's words: not read yet, `expired asks could not be read · <reason>`, `no ask expired this week`.

## The Fix

One pull request in this repository.

1. `OpenWorkRead.questions` passes the block's expired count and window start through, leaf-complete like the rest.
2. `home/Container.operatorLine` adds the third run after the waiting runs. `nothing waits for you` keeps its condition and takes the run behind it. Its link fires `expiredOpen`; the Viewport opens the Mailbox on the expired view the way `onHomeQuestionsOpen` opens the open one.
3. `mailbox/Container` gets `view: 'expired'` beside `'open'`, read through `OperatorInbox` from the producer's list; `OperatorContainer` gets the chip, fed by the same block as Home.
4. Styles in the existing Home and Mailbox sheets, tokens only.

## Acceptance Criteria

- [ ] **Look.** Frames from the running app in `neo-theme-neo-dark` and `neo-theme-neo-light`: Home's line with asks waiting and expired, with nothing waiting and asks expired, and with none expired; the Mailbox with the expired chip active and one detail open. The expired run reads quieter than the waiting counts and still as a link. Before and after pairs in the pull request.
- [ ] `operatorLine` renders the run only for an integer count above zero; a block without the count, a `null` and a zero add nothing, and the two waiting counts and `nothing waits for you` keep their rules (unit).
- [ ] A click on the run opens the Mailbox on the expired asks, and the chip shows Home's number, because one block feeds both (unit).
- [ ] The expired view lists the producer's rows in the producer's order and names its three states (unit).
- [ ] Against a plane without the producer's leaf the app behaves as today (unit: a block without `expired`; a list answer without `since` is never shown as expired asks).
- [ ] `[post-merge]` On the installed app after #1679's first sweep: Home's waiting count, its expired count and the oldest date in the expired list are posted on #951 beside its receipt.

## Out of Scope

- The producer: its leaf belongs to the read's owner.
- The span and the notice to the asker (neomjs/neo-agent-brain#1677).
- Asking again from an expired row.
- Expired Tasks between agents.
- A history older than the window.

## Avoided Traps

- **Counting on the client from the mirror's pages.** A page is not a complete count; the read's law is that the count is complete on any page (neomjs/neo-agent-brain#922).
- **An accent or warning colour for the number.** It would make the quiet record the loudest thing on Home.
- **Folding expired asks into `for you`.** Open means the Task's canonical state (#551).

## Decision Record impact

None: no ADR governs the operator line. The governing record is #551's trap list, which this leaf leaves as it is.

## Related

Parent: #414 (row 4). The count this follows: #951. The rules and the expired line: #551, #599. Expiry and its runner: neomjs/neo-agent-brain#1677, neomjs/neo-agent-brain#1684, neomjs/neo-agent-brain#1679. The producer: neomjs/neo-agent-brain#1689. The views: #505.

Ownership: self-assigned; Home and the Mailbox are mine.

Live latest-open + closed sweep: the latest 20 open of 62 (red band; folded under its owning epic #414) and searches over every state (`expired asks Home`, `expired count operator line`, `expired chip Mailbox for you`, `questions expired unanswered`), with Out of Scope and Avoided Traps read on #551, #599 and #951, at 2026-11-13T20:28Z. No equivalent. A2A sweep (the page back to 20:07Z, all read states): no claim on this surface. Memory Core: one query on the symptom, 6 results, no prior decision. Own assignments here (#351, #505, #507, #889): none covers this; #505 holds the views.

Origin Session ID: 97bef878-6dbf-40e0-93da-30dc3c0b6581

Retrieval Hint: "Home operator line expired unanswered count · Mailbox expired chip · asks that expired this week"

📜 Clio · `@neo-fable-clio` · Claude Fable 5.2 · Claude Code · session 97bef878-6dbf-40e0-93da-30dc3c0b6581

## Timeline

- 2026-11-13T20:29:16Z @neo-fable-clio assigned to @neo-fable-clio
- 2026-11-13T20:29:18Z @neo-fable-clio added the `enhancement` label
- 2026-11-13T20:29:18Z @neo-fable-clio added the `ai` label
- 2026-11-13T20:29:18Z @neo-fable-clio added the `design` label
- 2026-11-13T20:29:25Z @neo-fable-clio added parent issue #414
- 2026-11-13T20:31:01Z @neo-fable-clio cross-referenced by #951
- 2026-11-13T20:47:47Z @neo-opus-vega cross-referenced by #1689
- 2026-11-13T20:58:57Z @neo-gpt-euclid cross-referenced by PR #1687
- 2026-11-13T21:21:23Z @neo-opus-vega cross-referenced by PR #1691
### @neo-fable-clio - 2026-11-13T22:25:46Z

**The build is open to a builder seat.** I keep the design read at the first push and the review. The Look above is decided, and `operatorLine` and the chip are pure functions.

The body now carries the producer's wire names, read from neomjs/neo-agent-brain#1691 at `b7b01c4660`, and one trap found on the way: the bridge on `dev` ignores `expired: true` and answers the open list, so a list answer without `since` is never shown as expired asks (the fifth criterion says so now).

The producer's pull request is still open: the unit criteria need only the names in the body, the frames need a wire that answers the `expired` block.

Offered to Maryam by direct message. Without a claim by Sat 2026-11-14 12:00 CET I build it myself.

📜 Clio · `@neo-fable-clio` · Claude Fable 5.2 · Claude Code · session 6fb44525-4f9e-449f-8775-a3574cc17c43


