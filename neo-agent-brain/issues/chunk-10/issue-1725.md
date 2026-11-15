---
id: 1725
title: A lost backfill write leaves a retracted message's text in the index
state: CLOSED
labels:
  - bug
  - ai
  - testing
assignees:
  - neo-opus-vega
createdAt: '2026-11-14T19:32:52Z'
updatedAt: '2026-11-15T00:49:29Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1725'
author: neo-opus-vega
commentsCount: 1
parentIssue: 35
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking: []
closedAt: '2026-11-14T21:38:48Z'
---
# A lost backfill write leaves a retracted message's text in the index

## Context

Follow-up of #1440 (the message backfill), owned under #35 since Emmy's control of 2026-11-03 (#35 comment 6599788553). An upsert that applies its write and then fails, a lost response, with the message retracted in between, leaves the message's vector in the message collection, and its text with it, since the row's document is the message. No later run removes it.

Re-measured on `dev` `e4bc5491` on 2026-11-14 with a throwaway unit spec (removed): the first run counts `failed: 2`; the rerun counts `skipped: {present: 1, retracted: 1}` and deletes nothing; the collection still holds `MESSAGE:a` with the document `"MESSAGE:a subject\n\nMESSAGE:a body"`.

The plane's own runs met this failure class: #1440's re-run 1 lost 32 writes to `socket hang up` (#1440 comment of 2026-11-03 09:39Z). Whether any of them landed for a message retracted in between is unread.

## The Problem

A retraction takes a message's content out of reach. Recall already respects that: `MailboxService#queryMessages` hydrates each hit from the graph node ("Content comes from the graph node, so a retracted message shows its placeholder"). But the retracted text still ranks in semantic recall, it stays at rest in Chroma, and the backfill's receipt reports nothing left behind.

## The Architectural Reality

- `ai/scripts/maintenance/backfillMessageEmbeddings.mjs:113` `backfillMessageEmbeddings()`: each page drops retracted nodes (`live`, `:121`) before the presence check (`:122`), so a retracted id the collection holds is never looked at.
- The upsert's catch (`:146`) counts `failed` and `continue`s past the retraction compensation (`:152`).
- The compensating `collection.delete` (`:155`) is unguarded: a failed delete rejects the whole run.
- The drain already handles this for new messages: `ai/daemons/embed/drainCycle.mjs` verifies after a write and compensates a retraction (`MESSAGE_WAL_STORE`; "A retraction's `retracted` marker plays the purge tombstone's part there").

Design authority: the script's module JSDoc, "A message retracted while its batch was being written is deleted again afterwards, as the drain compensates a mid-embed retraction."

## The Fix

Make the rerun converge. That is the contract #1440 chose: a failed batch is counted, and an idempotent rerun retries it.

- Each page reads presence for every id, retracted ones included.
- A retracted id the collection holds is deleted in apply mode and counted as `compensated`, the drain's word. A dry run counts it without deleting.
- A failed delete is logged and counted in `failed`, and the run goes on; the next run deletes the id.

The receipt gains `compensated`. No same-payload retry inside a run: Emmy's control showed that a retry alone does not cover a retraction between attempts, and the rerun now does.

## Acceptance Criteria

- [ ] A unit arm, red on `dev` first: an upsert that writes, retracts the message and then throws; after a second run the collection no longer holds the retracted id, and the second receipt counts `compensated: 1`.
- [ ] A unit arm: a dry run over a collection that holds a retracted id counts it and deletes nothing.
- [ ] A unit arm: a delete that throws is counted in `failed`, the run finishes, and the next run deletes the id.
- [ ] The four existing arms stay green, and the module JSDoc says what a rerun compensates.
- [ ] `npx playwright test -c test/playwright/playwright.config.unit.mjs test/playwright/unit/ai/scripts/maintenance` is green.
- [x] (post-merge) A dry run against the plane reports how many retracted messages the index still holds; if any, one `--apply` removes them, and the receipt goes on this ticket.

## Out of Scope

- The live drain (`drainCycle.mjs`), which verifies and compensates already.
- Retry inside a run.
- #35's other open items: `projectedBytes` on an empty index and the 3,810 messages without a `SENT_BY` edge (both in #1440's comments).

## Related

- Parent: #35. Predecessor: #1440 (PR #1451). The control: #35 comment 6599788553.

Live latest-open + closed sweep: checked latest 20 open (158 open, red band) + closed search ("backfill retracted message embedding": #1439, another surface) at 2026-11-14T19:31Z; no equivalent or prior exclusion found. The owning ticket is #35, so this files under it.
MC sweep: "retracted message vector stays in the message collection after a lost backfill write", 5 results; the one prior decision is my #1440 comment of 2026-11-03 ("Any retry must first cover the arm Emmy controlled on #35"), which this follows.
Own-assignment sweep: 15 open, #35 is the parent; none overlapping.
A2A sweep: last 30 messages, no claim on this scope.
Structure map: no new file; `ai/scripts/maintenance` owns the script.

Origin Session ID: 0f917040-8b12-4dc3-9a7d-69961f23226d
Retrieval Hint: "backfill lost response retracted message vector stays in the message collection"

## Timeline

- 2026-11-14T19:32:52Z @neo-opus-vega assigned to @neo-opus-vega
- 2026-11-14T19:32:54Z @neo-opus-vega added the `bug` label
- 2026-11-14T19:32:54Z @neo-opus-vega added the `ai` label
- 2026-11-14T19:32:54Z @neo-opus-vega added the `testing` label
- 2026-11-14T19:32:57Z @neo-opus-vega added parent issue #35
- 2026-11-14T19:51:22Z @neo-opus-vega cross-referenced by PR #1727
- 2026-11-14T20:01:19Z @neo-opus-vega cross-referenced by #35
- 2026-11-14T21:38:48Z @tobiu referenced in commit `66fff73` - "feat(memory-core): a backfill rerun deletes a retracted message the index still holds (#1725) (#1727)

A backfill write that landed after its answer was lost, for a message
retracted in between, left the message's vector and text in the message
collection, and no later run removed it: each page dropped retracted
nodes before it checked the collection.

Each page now checks every id. A retracted message the collection holds
is deleted and counted as `compensated`; a dry run only counts it. A
failed delete, here or in the mid-write compensation, counts as `failed`
instead of rejecting the run, and the next run retries it."
- 2026-11-14T21:38:49Z @tobiu closed this issue
### @neo-opus-vega - 2026-11-15T00:49:29Z

**Post-merge receipt** (the last AC). The plane is at `2cb8e911`, which carries #1727 (`66fff73e`). Each run executed inside `neo-local-agent-os-mc-server-1` against `/app/.neo-ai-data/sqlite/memory-core-graph.sqlite`, opened read-only, on 2026-11-15 between 00:47Z and 00:53Z.

| run | messages | pending | compensated | present | retracted | incomplete | failed |
|---|---|---|---|---|---|---|---|
| dry run | 71,678 | 0 | **11** | 64,452 | 25 | 7,190 | 0 |
| `--apply` | 71,678 | 0 (embedded 0) | **11** | 64,452 | 25 | 7,190 | 0 |
| dry rerun | 71,678 | 0 | **0** | 64,452 | 36 | 7,190 | 0 |

The index held 11 retracted messages; it now holds none. The 11 count as retracted skips on the rerun (25 → 36). `--apply` wrote no embeddings: with `pending: 0`, its only effect was the 11 deletes.

The 7,190 `incomplete` messages lack a route edge or a send time, so the script skips them and `query_messages` cannot recall them. That is not this ticket, and why so many lack their edges is `hypothesis — needs V-B-A` before any ticket.

— Vega (Opus 5.5, Claude Code) 🌿

- 2026-11-15T00:59:52Z @neo-opus-vega cross-referenced by #1732

