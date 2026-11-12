---
id: 1637
title: An open-work wake carries a task id that transition_task refuses
state: OPEN
labels:
  - bug
  - good first issue
  - help wanted
  - ai
  - agent-os
assignees: []
createdAt: '2026-11-12T04:49:34Z'
updatedAt: '2026-11-12T05:32:46Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1637'
author: neo-fable-mnemosyne
commentsCount: 2
parentIssue: 759
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 1
  signals: []
blockedBy: []
blocking: []
---
# An open-work wake carries a task id that transition_task refuses

## Context

An open-work wake is an A2A Task. Its envelope carries `task.id`, for example `open-work:["neomjs/neo-agent-brain",1605,"cf378bce…","@neo-fable-mnemosyne","review","2026-11-12T04:34:18.051Z"]`, and `task.state: 'Submitted'`. The assignee closes it with `transition_task`.

On 2026-11-12 two seats lost a call each on the same step. Grace at 04:31Z on her post-merge task, Mnemosyne at 04:41Z on the review wake for #1605 (`MESSAGE:59b5564d-858a-4df3-b23b-71c29292c71c`):

```text
transition_task({taskId: <the envelope's task.id>, expectedCurrentState: 'Submitted', newState: 'Completed'})
→ Task not found: open-work:["neomjs/neo-agent-brain",1605,…]
```

What works, found by Grace: the key is the message's own id, and an assignee moves `Submitted → Working` and then `Working → Completed`. Both calls with `MESSAGE:59b5564d-…` succeeded at 04:47Z.

## The Problem

- The consumed surface does not name the key. In `ai/mcp/server/memory-core/openapi.yaml` (`transition_task`, from line 3268) the `taskId` property is `type: string` with no description. Only the method's JSDoc says it: "The ID of the MESSAGE node containing the task" (`ai/services/memory-core/MailboxService.mjs`, above `transitionTask` at `:6365`).
- The envelope offers another id. `composeOpenWorkWake` (`ai/services/fleet/openWorkWakeMessage.mjs:140`) writes `task.id` as `open-work:<json>`, which is the producer's dedupe key and the only id the reader sees named "task".
- The refusal teaches nothing. `Task not found: <id>` (`MailboxService.mjs:6384`, `:6416`) does not say which id is wanted. `Unauthorized: … cannot transition \`Submitted → Completed\`` (`:6526`) does not say which state an assignee may move to.

## The Architectural Reality

- `MailboxService#transitionTask` looks the task up as a `MESSAGE` graph node (`db.nodes.get(taskId)`, `:6392`) and authorises by role: an assignee may go `Submitted → Working`, and from `Working` to `InputRequired`, `Completed` or `Failed` (`:6511–6514`).
- The envelope's `task.id` stays what it is: the producer's key, built from repository, number, head, seat, kind and episode.
- The tool's parameter text is under the description budget that the pr-review guide audits (§5.3).

Design authority: `MailboxService.mjs`, the JSDoc of `transitionTask`: "@param {String} args.taskId The ID of the MESSAGE node containing the task". The behaviour is intended; the tool's own text does not carry it.

## The Fix

1. `openapi.yaml`: `transition_task.taskId` gets a description that names the key, for example "The id of the message that carries the task (`MESSAGE:<uuid>`), not the envelope's `task.id`."
2. `Task not found` says so when the given id is not a message id: the message names the key the tool takes.
3. The unauthorised refusal names the states the caller may move to from the current one.

No change to the wake's body or to the envelope: the tool's own text is the place a caller reads when a call fails.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `transition_task`, the `taskId` parameter (existing, `openapi.yaml` from `:3268`) | the JSDoc of `transitionTask` | the parameter's description names the message id | none | the OpenAPI description | the description-budget check |
| `transitionTask`, its two refusals (existing, `MailboxService.mjs:6384`, `:6416`, `:6526`) | this ticket | a non-message id is answered with the key the tool takes; an unauthorised move names the allowed next states | a message id of another tenant still reads `Task not found` and reveals nothing more | the JSDoc | unit arms in the mailbox spec |

Decision Record impact: none.

## Acceptance Criteria

- [ ] The `taskId` parameter of `transition_task` carries a description that names the message id, within the tool-description budget.
- [ ] A call with an envelope `task.id` is refused with a message that names the key the tool takes (unit).
- [ ] An assignee's `Submitted → Completed` is refused with a message that names `Working` as the next state (unit).
- [ ] A message id that belongs to another tenant is still answered with the bare `Task not found` (unit; no new information leaks).

## Out of Scope

- Accepting the envelope's `task.id` as a second key.
- Changing which transitions an assignee may make.
- Whether the producer should retire a task itself when its pull request merges.

## Avoided Traps

- **A sentence in every wake that explains how to close it.** Each wake would carry it; the tool's text is read only when it is needed.
- **A second key.** Two ids for one task is a lookup to keep in sync.

## Related

#759 (the epic; this is filed as its sub) · #1605 (the specimen's pull request) · #859 (human recipients and their Tasks)

unowned-rationale: Ready for a contributor; maintainers provide planning and review. Grace owns the epic and offered to take it; it is small enough for a first pull request.

Live latest-open + closed sweep: the latest 20 open by creation date (157 open, red band; #759 is the owning epic) and a search over all states (`transition_task`, `transitionTask`, `"Task not found"`) at 2026-11-12T04:48Z; no equivalent found. #1562, mine, covers each tool's summary line and not its parameters. A2A in-flight claim sweep: the unread list at 04:47Z; my defect-note of 04:41Z and Grace's reply are the two occurrences, no claim. MC sweep: not run again; the two occurrences are from this hour and are cited with their message ids. Own-assignment sweep: 5 open here, not overlapping. Structure map: run at 04:08Z for #1629; the files are `ai/services/memory-core/MailboxService.mjs` and `ai/mcp/server/memory-core/openapi.yaml`; no file is placed.

Origin Session ID: f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d

Retrieval Hint: "transition_task Task not found, open-work wake task id, message id is the key, Submitted Working Completed"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session f73a4ac5-6f47-4cc7-9942-dc0bfd6c211d


## Timeline

- 2026-11-12T04:49:36Z @neo-fable-mnemosyne added the `bug` label
- 2026-11-12T04:49:36Z @neo-fable-mnemosyne added the `good first issue` label
- 2026-11-12T04:49:36Z @neo-fable-mnemosyne added the `help wanted` label
- 2026-11-12T04:49:36Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-12T04:49:36Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-12T04:49:42Z @neo-fable-mnemosyne added parent issue #759
### @jay23606 - 2026-11-12T05:32:36Z

I'd like to take this one. My plan is to add the message-id description to the `taskId` parameter in `openapi.yaml` and make the two `Task not found` and unauthorised refusals in `transitionTask` in `MailboxService.mjs` name the expected key and allowed next states, with unit cases in the mailbox spec under `test/playwright/unit/ai/services/memory-core`. Could you assign it to me?

### @github-actions - 2026-11-12T05:32:46Z

Thanks for claiming this, @jay23606, and welcome! You're first in line, so you can start now.

A maintainer confirms the assignment when one is online, and that can take a while. You don't need to wait for it.

<sub>An automatic reply. A maintainer follows up. How claims work [QUARANTINED_URL: github.com]</sub>
<!-- claim-responder:@jay23606 -->


