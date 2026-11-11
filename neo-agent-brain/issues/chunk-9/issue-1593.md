---
id: 1593
title: 'IdentityFirewall.md links the Engine''s AGENTS.md pointer, not the rules'
state: OPEN
labels:
  - documentation
  - good first issue
  - ai
assignees: []
createdAt: '2026-11-11T03:07:09Z'
updatedAt: '2026-11-11T09:59:27Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1593'
author: neo-opus-grace
commentsCount: 1
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
# IdentityFirewall.md links the Engine's AGENTS.md pointer, not the rules

## Context

neomjs/neo#20749, the Engine half of neomjs/neo-agent-skills#335, makes the Engine's `AGENTS.md` a pointer in git. Every installed checkout holds the generated rules in that file, while GitHub shows only a three-line note saying the file is generated. Once it merges, the link at `learn/agentos/IdentityFirewall.md:153`, `https://github.com/neomjs/neo/blob/dev/AGENTS.md`, lands on that note instead of the rules the guide describes.

## The Problem

A reader of the Identity Firewall guide who follows the link to the rules finds a pointer to them.

## The Architectural Reality

Design authority: n/a — the link's target changes what it shows; no behavior is called wrong.

- The rules' source is https://github.com/neomjs/neo-agent-skills/tree/dev/agents-md: the preamble files plus `sections/`, composed by the `neo-agent-skills-agents-md` bin.
- The Engine moved its own two links the same way in neomjs/neo#20749 (`learn/blog/ai-native-platform-answers-questions.md:18`, `learn/blog/context-engineering-done-right.md:1880`).

## The Fix

Point the link on `learn/agentos/IdentityFirewall.md:153` at https://github.com/neomjs/neo-agent-skills/tree/dev/agents-md, and say in the sentence that each checkout's `AGENTS.md` is generated from there.

## Acceptance Criteria

- [ ] `learn/agentos/IdentityFirewall.md` no longer links `neomjs/neo/blob/dev/AGENTS.md`, and its link reaches the `agents-md` directory of neomjs/neo-agent-skills.
- [ ] The PR's checks pass.

## Out of Scope

- The Engine's links (neomjs/neo#20749) and any other wording in the guide.

## Related

- neomjs/neo-agent-skills#335 (source), neomjs/neo#20749

Live latest-open + closed sweep: checked the latest open at 2026-11-11T02:57Z and searched titles for `AGENTS.md` (#644 and #136, neither about this link); no equivalent. Backlog band: 156 open (red); no owning ticket covers this link, and it is the Brain's part of #335's AC-6.
A2A in-flight claim sweep: the last 30 messages carry no claim on `IdentityFirewall.md`.
Memory Core rationale sweep: #335's thread holds the decision to generate the file; nothing keeps this link.
Own-assignment sweep: nothing assigned to me here touches the guide.
Structure map: N/A. A one-line documentation change; no placement is decided.

unowned-rationale: Ready for a contributor; maintainers provide planning and review. It is one line with public context only, and it can land before or after neomjs/neo#20749.

Retrieval Hint: "IdentityFirewall AGENTS.md link pointer agents-md sections"

Origin Session ID: 40a9201b-5e9e-4845-ab9f-dee27312077d


## Timeline

- 2026-11-11T03:07:11Z @neo-opus-grace added the `documentation` label
- 2026-11-11T03:07:11Z @neo-opus-grace added the `good first issue` label
- 2026-11-11T03:07:12Z @neo-opus-grace added the `ai` label
- 2026-11-11T03:07:35Z @neo-opus-grace cross-referenced by #335
- 2026-11-11T03:07:38Z @neo-opus-grace cross-referenced by PR #20749
- 2026-11-11T03:48:52Z @neo-opus-grace cross-referenced by #1594
- 2026-11-11T03:49:27Z @neo-opus-grace changed title from **IdentityFirewall.md links an AGENTS.md the Engine no longer tracks** to **IdentityFirewall.md links the Engine's AGENTS.md pointer, not the rules**
### @erSpell - 2026-11-11T09:59:27Z

Working on this!

- 2026-11-11T12:20:01Z @erSpell cross-referenced by PR #1602

