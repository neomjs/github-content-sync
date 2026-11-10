---
id: 1569
title: 'Read the call counts four weeks after the trigger lines land: did saying when move use?'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-11-10T02:34:08Z'
updatedAt: '2026-11-10T02:34:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/1569'
author: neo-fable-mnemosyne
commentsCount: 0
parentIssue: 1562
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy:
  - '[ ] 1567 Knowledge Base and file-system servers: each tool line says what the tool does and when to use it'
  - '[ ] 1565 Neural Link: each of its 60 tool lines says what the tool does and when to use it'
  - '[ ] 1564 Memory Core: each of its 55 tool lines says what the tool does and when to use it'
blocking: []
---
# Read the call counts four weeks after the trigger lines land: did saying when move use?

## Context

The measurement leaf of #1562 (graduated from neomjs/neo#20680; the open part of OQ-7). Blocked by the server leaves. Due four weeks after the last of them merges.

## The Problem

The epic rests on one untested sentence: a tool is called where its line says when to use it. Before the lines, in the 7 days to 2026-11-10T00:08Z, Memory Core served 271,805 calls, 37 of its 55 tools were called and 18 were not. If the same tools stay uncalled after their lines say when, the lines were not the cause, and the next step is the catalog or a rule that names the tool.

## The Architectural Reality

- Memory Core, the Knowledge Base and the Neural Link record calls per tool (`get_memory_core_tool_metrics` and the two recorder services); the GitHub workflow server does not.
- Most calls come from daemons, hooks and the Fleet Manager, which read no listing. The comparison is about the tools a model chooses.

## The Fix

One table on #1562: per tool, calls in the 7-day baseline and in the same 7-day window four weeks after the lines, for the three servers that record, with the callers that read no listing set apart. Then a verdict in one paragraph: which tools moved, which did not, and what that says about the sentence above.

## Acceptance Criteria

- [ ] The table covers every tool of Memory Core, the Knowledge Base and the Neural Link, with its line before and after.
- [ ] The 18 Memory Core tools without a call in the baseline are listed with their count after.
- [ ] Calls by daemons, hooks and the Fleet Manager are separated or the table says why they could not be.
- [ ] The verdict names one next step, or says the lines did their job.

## Out of Scope

Changing a line or a tool. Call recording on the GitHub workflow server.

## Related

#1562 (parent), the three server leaves that record calls (they block this one), #1555.

unowned-rationale: it starts four weeks after the server leaves; for the seat that then holds #1562's closeout. No `good first issue`: it reads private usage data.

Live latest-open + closed sweep and A2A, MC and own-assignment sweeps: as on #1562 at 2026-11-10T02:27Z; this leaf is filed with it and has no equivalent.

Origin Session ID: fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

Retrieval Hint: "tool call counts before and after trigger lines measurement"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session fda09208-5d6a-4fc7-a0d4-69a53fdf2dec

## Timeline

- 2026-11-10T02:34:09Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-10T02:34:10Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-10T02:34:10Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-10T02:34:10Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-10T02:34:46Z @neo-fable-mnemosyne added parent issue #1562
- 2026-11-10T02:35:10Z @neo-fable-mnemosyne marked this issue as being blocked by #1564
- 2026-11-10T02:35:11Z @neo-fable-mnemosyne marked this issue as being blocked by #1565
- 2026-11-10T02:35:13Z @neo-fable-mnemosyne marked this issue as being blocked by #1567

