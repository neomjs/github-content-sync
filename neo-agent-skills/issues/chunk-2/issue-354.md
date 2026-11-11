---
id: 354
title: The single-subagent boundary no longer states the policy in force
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
  - needs-design-authority
assignees: []
createdAt: '2026-11-11T20:18:55Z'
updatedAt: '2026-11-11T21:35:42Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/354'
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
---
# The single-subagent boundary no longer states the policy in force

## Context

The swarm-topology anchor says, for every maintainer in five repositories: fan-out and Workflows are forbidden, and "A SINGLE tactical subagent is permitted ONLY on the operator's explicit in-session permission (rare)" (`agents-md/sections/1400-swarm-topology-anchor.md:25`).

On 2026-11-11 the operator decided, after a measured pilot on a Claude Code seat: "we can allow claude peers to use one subagent at a time for use cases that fit." The same day he confirmed that GPT and Gemini seats have worked under a standing authorization for bounded helpers for months. So the loaded sentence states the policy of no family.

## The Problem

- A seat loads a rule it does not work under. Where a seat's own memory file says otherwise, it carries two rules that contradict each other every turn.
- The sentence gives a legitimate use no shape, so each use is argued anew, and the old failure has no guard: several helpers at once, on the seat's own model.
- The pilot, three runs with one helper each, measured from the helpers' transcripts:

| | One small read | A sweep of 56 files (309 KB), 40 answers with a cited line each |
|---|---|---|
| Helper type | general-purpose | read-only search (`Explore`) |
| Calls to the mailbox, the Memory Core or GitHub; file writes | none | none, on both models tried |
| Context at entry, before any work | 68,662 tokens | 39,617 |
| Cited lines checked by script | 2 of 2 | 40 of 40 on the mid-tier model, 94 of 94 on the small one |
| Added to the seat's context | about 3k tokens | about 4k, against about 90k for the same read inline |

What it showed:
- A helper pays for a bounded read whose raw material the seat does not need to judge. It does not pay for a small read: the entry load is larger than the file.
- A helper with no named model runs on the seat's model.
- The general-purpose type carries the seat's instruction file and memory index in its context. The read-only search type carried neither and entered 29k tokens lower.
- The two models agreed on 35 of 40 answers. The small one read one question more widely than it was meant; where a helper must judge what a question means, the mid-tier model was the better reader.

## The Architectural Reality

- The sentence is turn-loaded for the `maintainer` audience in `neo`, `neo-agent-brain`, `neo-agent-skills`, `neo-agent-institution` and `devindex` (the section's `repos:` line). The section is 4,114 bytes; a generated file may not pass 23,800.
- One rule of the engine's settings template is read and still gates a helper: `permissions.ask` holds `Agent` and `Task` (`neomjs/neo`, `.claude/settings.template.json:57-60`), so a helper call asks the operator. Whether that prompt stays is neomjs/neo#20795. The same file restates the old sentence in `autoMode.hard_deny` (`:80`), which nothing reads: Claude Code takes no `autoMode` from project settings (Vega's measurement, neomjs/neo#20794 comment 6807353591), and neomjs/neo#20796 deletes that block. The classifier reads this anchor as part of the instruction file. The settings also deny the Workflow tool.
- Design authority: the operator's decision of 2026-11-11, quoted above.
- Decision Record impact: none. The compaction taxonomy (ADR 0007 in `neo-agent-brain`) governs the shape of the edit.

## The Fix

Replace the single-subagent sentence with the rule in force, written as what a seat does:

> One read-only helper at a time is allowed for a bounded read whose raw material the seat does not need to judge. Name its model: the small one for a sweep a script can check, the mid-tier one where it must judge what a question means. Use the read-only search type. Never fork. The helper writes nothing: no GitHub, no mailbox, no memory, no file. The seat checks every line it takes from the report. Whatever needs an owner, a signature or continuity goes to a peer.

The ban on fan-out and Workflows stays as it is.

## Contract Ledger

| Target surface | Source of authority | Proposed behaviour | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| the Boundary sentence of `1400-swarm-topology-anchor.md`, loaded every turn | the operator's decision of 2026-11-11 | one read-only helper at a time, model named, never a fork, nothing written, the seat checks cited lines | anything outside that shape still needs the operator's word in that session | the sentence itself | the pilot table above; a fresh-session check after the merge |
| the ask-gate for `Agent` and `Task` in the engine's `.claude/settings.template.json` | the operator's decision on neomjs/neo#20795 | the prompt stays (one helper per confirmation) or the two entries leave | while the gate holds, every helper call asks the operator | neomjs/neo#20795 | the fresh-session check |

## Acceptance Criteria

- [ ] The anchor states the rule above for Claude Code seats, in no more bytes than the sentence it replaces.
- [ ] The sentence no longer claims one policy for every family. It either scopes itself to Claude Code or carries each family's rule; one seat of each family reads the pull request and says which.
- [ ] `npm run lint` and `npm test` pass, and `neo-agent-skills-agents-md --repo neo` stays under 23,800 bytes.
- [ ] The pull request body names the version bump (minor: a rule changes).
- [ ] After the merge, a fresh Claude Code session that loads the new text, asked for a sweep of 50 files, starts one read-only helper with a named model; asked to read one file, it reads the file itself. The transcript is the receipt. If the ask-gate of neomjs/neo#20795 still holds, the check records the prompt and passes once the operator confirms it.

## Out of Scope

A hook on the helper call that checks the three fields (a named model, no fork, one at a time): worth a leaf of its own in the repository that owns the seat hooks. A helper as the cold reader of a design: not measured yet. Any budget or price.

## Avoided Traps

- **Leaving the text and trusting each seat's memory.** That is today's state.
- **"Use a helper when it is useful."** Without a shape, the old failure returns.
- **A ban with exceptions.** The new sentence says what to do; what is outside it keeps the old rule.

## Related

#349 (another loaded text that names what no longer exists). neomjs/neo#20795 (the ask-gate for a helper call) and neomjs/neo#20796 (the dead `autoMode` block).

unowned-rationale: Ready for a maintainer seat. It edits a turn-loaded anchor, so it is not a good first issue; one seat of each family reads the pull request.

Live latest-open + closed sweep: latest 20 open of 35 (orange band; no open ticket or epic owns this sentence) and a search over all states (`subagent`, `helper`, `fan-out`, `swarm topology`) at 2026-11-11T20:20Z; ten hits read by title, none on this boundary; closed #49 (seat budget discipline) is adjacent and does not cover it. A2A in-flight claim sweep: the newest unread and read messages at 20:20Z; no claim on this surface. MC sweep: six queries on helper use by Claude, GPT and Gemini seats on 2026-11-11; the operator's earlier words and three seats' records were found, no decision that keeps the sentence. Own-assignment sweep: 1 open here (#191), a different surface. Structure map: N/A, Brain-hosted; nothing is placed. Meta-skill sweep: the section is not a skill; no router changes.

Origin Session ID: f51f7694-6d97-405f-b57b-b269152a5ebd

Retrieval Hint: "one read-only helper at a time for Claude seats: swarm topology anchor boundary sentence, named model, never a fork"

🪢 Mnemosyne (Claude Fable 5.2, Claude Code) · session f51f7694-6d97-405f-b57b-b269152a5ebd

## Timeline

- 2026-11-11T20:18:57Z @neo-fable-mnemosyne added the `enhancement` label
- 2026-11-11T20:18:57Z @neo-fable-mnemosyne added the `ai` label
- 2026-11-11T20:18:57Z @neo-fable-mnemosyne added the `model-experience` label
- 2026-11-11T20:18:57Z @neo-fable-mnemosyne added the `agent-os` label
- 2026-11-11T21:15:30Z @neo-opus-grace cross-referenced by #20794
- 2026-11-11T21:21:46Z @neo-fable-mnemosyne cross-referenced by #20795
- 2026-11-11T21:22:21Z @github-actions added the `needs-design-authority` label

