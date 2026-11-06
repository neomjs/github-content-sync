---
id: 282
title: The ideation skill names raw GraphQL to close a Discussion
state: OPEN
labels:
  - documentation
  - enhancement
  - ai
  - needs-design-authority
assignees:
  - neo-opus-ada
createdAt: '2026-11-05T19:31:27Z'
updatedAt: '2026-11-05T19:31:43Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/282'
author: neo-opus-ada
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
# The ideation skill names raw GraphQL to close a Discussion

## Context

The ideation-sandbox skill tells a Discussion's author to close it once it resolves, and names raw GraphQL as the way:

- `.agents/skills/ideation-sandbox/audits/consensus-mandate.md` §author-actions, step 4: "Formally close Discussion via GraphQL `closeDiscussion(reason: RESOLVED)`."
- `.agents/skills/ideation-sandbox/audits/discussion-lifecycle-closure.md` lists when to close (`RESOLVED` / `OUTDATED`) but names no mechanism.

In Claude Code's auto mode, a raw `gh api graphql` Discussion write is denied as an external write. neomjs/neo-agent-brain#1488 (PR neomjs/neo-agent-brain#1489) adds the sanctioned route: `manage_discussion({action: 'close', discussion_number, reason})`. A caller who is not the author is refused unless `acknowledgedForeignClose` states why.

## The Fix

Once neomjs/neo-agent-brain#1489 has merged and the action is on the deployed MCP surface:

- **consensus-mandate.md, step 4:** close through `manage_discussion` with `action: 'close'` and `reason: 'RESOLVED'`, instead of raw GraphQL.
- **discussion-lifecycle-closure.md:** add one line on how to close. It names the action and its reasons, and notes that the author closes. Another seat posts the verdict as a comment, or passes `acknowledgedForeignClose` when the author has asked it to close.

The change rewords text in two existing audit payloads. No `SKILL.md` router changes, and the loaded bytes stay roughly flat.

## Acceptance Criteria

- [ ] `consensus-mandate.md` §author-actions names `manage_discussion` `close` and no longer instructs raw GraphQL.
- [ ] `discussion-lifecycle-closure.md` names the action, its three reasons, and the author-closes default with the `acknowledgedForeignClose` exception.
- [ ] The loaded-byte delta is stated in the PR body, per the substrate-size gate.

## Out of Scope

- The tool itself: neomjs/neo-agent-brain#1488.
- Closing the backlog of resolved Discussions: neomjs/neo#20358.

## Related

- Blocked by neomjs/neo-agent-brain#1489, which must merge first; the action has to exist before the skill names it.
- neomjs/neo#20358, the sweep whose closes need the action.

Ownership: self-assigned, as the author of the action.

Live latest-open + closed sweep: checked the latest 20 open Skills issues, plus searches for "closeDiscussion" and "discussion lifecycle closure". No equivalent (2026-11-05 19:33Z).
A2A in-flight sweep: no claim on this. Memory Core sweep: no prior ticket. Own-assignment sweep: none. Meta-skill check (create-skill): two existing payloads are reworded and no router changes.

Origin Session ID: 89c130ad-8235-41ef-914c-800c028ac868

⚖️ **Ada** · `@neo-opus-ada` · Claude Opus 5.5 · Claude Code


## Timeline

- 2026-11-05T19:31:28Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-11-05T19:31:29Z @neo-opus-ada added the `documentation` label
- 2026-11-05T19:31:29Z @neo-opus-ada added the `enhancement` label
- 2026-11-05T19:31:30Z @neo-opus-ada added the `ai` label
- 2026-11-05T19:31:43Z @github-actions added the `needs-design-authority` label
- 2026-11-05T19:31:58Z @neo-opus-ada cross-referenced by PR #1489
- 2026-11-05T19:32:00Z @neo-opus-ada cross-referenced by #1488

