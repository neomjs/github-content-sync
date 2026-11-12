---
id: 1618
title: The mirror misses a wrap-up ask that opens with an unlisted verb
state: CLOSED
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees:
  - neo-opus-ada
createdAt: '2026-11-11T22:21:28Z'
updatedAt: '2026-11-12T12:55:05Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1618'
author: neo-fable-mnemosyne
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
closedAt: '2026-11-12T12:55:05Z'
---
# The mirror misses a wrap-up ask that opens with an unlisted verb

## Context

PR #1529 (#78) taught the Stop hook's deference mirror to read the Claude desktop app's question box: the `ask` and `actions` of `mcp__ccd_turn__wrap_up`. On 2026-11-11 a turn stated a recommendation in its text and then asked the operator an A or B question in the box. The mirror stayed silent. The operator's note on that box:

> these A or B button responses are so far claude specific. most of the time you already have a recommended path. it gets more tricky in nightshifts, when you act while i am not around.

## The Problem

Reproduced on `dev` `c27e8b9b` by calling the module's own function:

```text
ask      "Replace the per-call prompt with a hook that checks every helper call?"
buttons  "Yes, the hook" | "Keep the prompt"
text     "... I recommend the third. With the prompt, a helper that saves a few minutes waits for you, and at night it never starts."
matchWrapUpAskDeference(ask, text)   -> null        expected: the ask

control  ask "File it?", text "I would want this on."            -> "File it?"
control  ask "Merge the pull request?", first action `merge`      -> null
```

1. **The verb list decides before the stated pick is read.** `AGENT_STEP_ASK` (`ai/scripts/lifecycle/deferencePhraseMatch.mjs:362`) admits an ask only when it opens with "should I", "want me to" or one of sixteen verbs. A question about a choice the agent has already made can open with any verb: replace, keep, use, switch, remove, add. The ask fails at `:412`, so `STATED_PICK` (`:342`), which matches "I recommend", is never consulted.
2. **At night the box has no reader.** The function reads both contexts the same way (its JSDoc, `:399-402`). A turn the operator did not prompt may therefore end on a box question, and the question waits for a click nobody is there to give. It is on no ticket unless the seat put it there.

## The Architectural Reality

- `laneStateStopHook.mjs:641` extracts the turn's last `wrap_up` call; `stopHookDecision.mjs:381-382` runs the text check first and the wrap-up check when no phrase matched.
- `OPERATOR_OWNED` (`:354`) carves the questions that are the human's: a merge, credentials, payments, his display, a design read. A first action of `merge` is carved as well (`:413`).
- By the box's own contract its first button is the agent's pick, so a box with buttons always carries a preference. A stated pick in the turn's text is the stricter test, and it exists.
- Design authority: the operator's decision of 2026-08-25 recorded on #78, case 3, "an equal peer chooses, and 99% of the time you already know what you want to do next", which fires also while he is in the loop; and his note of 2026-11-11 above.

## The Fix

1. In `matchWrapUpAskDeference`, a stated pick in the turn's text or in the ask arms the mirror for every ask that `OPERATOR_OWNED` does not carve, whatever verb opens it. An ask without any stated pick keeps today's verdict.
2. A question whose answer only the human has stays his: what he saw, which account, a merge. The table below is the contract.
3. In a turn the operator did not prompt, a `wrap_up` call that carries an `ask` blocks the stop once, with a directive that names the two exits: take the stated pick, or record the question on its ticket and continue with another lane.
4. The reminder for a box says the test in one sentence: a question is the human's only when no gate of his will see the change before it takes effect.

| ask | the turn's text | verdict |
|---|---|---|
| "Replace the per-call prompt with a hook that checks every helper call?" | "I recommend the third." | mirror |
| "Which fix first?" | "I would take the hook first." | mirror |
| "Keep the prompt?" | "My pick is the hook." | mirror |
| "Did the app show you a confirmation?" | "I recommend the third." | no mirror: a fact only he has |
| "Run one small helper now to record whether its call asks you first?" | no stated pick | no mirror, as today |
| "Merge the pull request?", first action `merge` | "I recommend merging." | no mirror, as today |

## Acceptance Criteria

- [ ] The reproduction above returns the ask.
- [ ] The six rows hold in `test/playwright/unit/ai/scripts/lifecycle/deferencePhraseMatch.spec.mjs`.
- [ ] In a turn the operator did not prompt, a `wrap_up` call with an `ask` blocks the stop once, and the directive names both exits; `stopHookDecision.spec.mjs` covers it.
- [ ] The existing carves still pass: operator-owned words, a `merge` first action, a quoted or attributed pick.

## Out of Scope

- Other harnesses. They have no question box; the surface every family shares for a question to the operator is its ticket.
- Whether the text check should fire on "I recommend X" followed by a question in plain prose.
- The night's turn supply: #95.

## Avoided Traps

- **Growing the verb list.** The next question opens with another verb.
- **Mirroring every box.** A question whose answer only the human has must pass.

## Related

#78 and PR #1529 (the wrap-up arm) · #136 (mechanical enforcement in place of prompt text) · #95 (the night's turn supply)

Decision Record impact: none. The governing decision is the operator's, recorded on #78.

handoff: @neo-opus-ada is asked, as the author of the wrap-up arm. Any maintainer seat can take it; it changes what ends a turn on every Claude seat, so it is not a good first issue.

Live latest-open + closed sweep: the latest 20 open (153 open in all, red band; no open ticket owns the mirror's matching, #78 is closed as completed, so this is filed as its follow-up) and a closed search (deference mirror wrap-up, "wrap-up ask", "question box", "next-step box"; #78 read, #137 read by title) at 2026-11-11T22:19Z; no equivalent or prior exclusion found. A2A in-flight claim sweep: the latest 12 messages of all read states at 22:19Z and the 30 before them at 21:14Z; no claim on this surface. MC sweep: "no need to ask A or B when you already know what you want to do next; a question in the next-step box that nobody answers at night", 6 results, no prior decision beyond #78's. Own-assignment sweep: 5 open here; #95 is adjacent (the wake policy) and does not cover the box. Structure map: run with `--files --loc`; both files exist under `ai/scripts/lifecycle/`, nothing is placed.

Origin Session ID: b3dee5fd-c7f4-45a9-8281-0c2face5b7b6

Retrieval Hint: "deference mirror wrap-up ask verb list; a stated recommendation followed by an A or B box; a box question at night"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session b3dee5fd-c7f4-45a9-8281-0c2face5b7b6

## Timeline

- 2026-11-11T22:21:29Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-11T22:21:29Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T22:21:30Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-11T22:21:30Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-11T22:22:32Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-12T01:16:36Z @neo-opus-ada cross-referenced by #1622
- 2026-11-12T06:32:08Z @neo-opus-ada cross-referenced by PR #1639
- 2026-11-12T06:38:15Z @neo-opus-ada cross-referenced by #1640
- 2026-11-12T12:55:04Z @tobiu referenced in commit `cfb998a` - "feat(lifecycle): the question box mirror reads any verb (#1618) (#1639)

A recommendation in the turn's text or in the ask ("I'd", "I recommend",
"my pick is") now arms the box whatever verb opens it. A bare "I'll"
keeps the agent-step gate: replayed over Ada's 402 dialogue asks, it
armed twenty questions that were the operator's own. A box asking what
he saw or a step only he takes stays his.

In a turn the operator did not prompt, any ask blocks once, with a
directive naming both exits: take the stated pick, or record the
question on its ticket and continue with another lane. Both box
reminders state the test: a question is the human's only when no gate
of his will see the change before it takes effect."
- 2026-11-12T12:55:06Z @tobiu closed this issue
- 2026-11-12T15:10:20Z @neo-gpt-euclid cross-referenced by PR #1645

