---
id: 186
title: Make the Brain README own the institution story
state: CLOSED
labels:
  - documentation
  - enhancement
  - design
  - ai
  - agent-os
assignees:
  - neo-gpt
createdAt: '2026-08-27T14:11:47Z'
updatedAt: '2026-08-28T07:29:14Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/186'
author: neo-gpt
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
closedAt: '2026-08-28T07:29:14Z'
---
# Make the Brain README own the institution story

## Context

After the repository split, the merged Brain README is 231 lines / 13.3KB, while the Engine README is 233 lines / 22.2KB and still carries a 52-line / 5.9KB Brain-specific section. Engine currently owns the named-maintainer roster, the night-shift/wake explanation, and the richest institution narrative; Brain has no roster and no night-shift section. The Engine therefore still explains the Brain better than the Brain explains itself.

Both READMEs also retain the retired outward product name, and both organization maps place **You are here** before the repository link. Operator direction requires repository text first and a plain suffix marker outside the URL.

## The Problem

The code and package metadata moved receive-first, but the public institution story did not. A cold reader in the canonical Agent OS repository gets mechanisms and install modes without the concrete institution that makes them meaningful: persistent named peers, wake-driven night work, cross-family review, and the human merge gate. Meanwhile the Engine carries that detail and the retired outward product vocabulary.

## The Architectural Reality

- Brain owns the Agent OS and the institution mechanism; Engine owns the Body and the organism-level apex.
- Agent Institution is the outward operator-product name. Fleet vocabulary remains valid only inside the subsystem per D#17247.
- Identity claims and roster facts must be verified from current canonical identity sources, not copied stale from Engine prose.
- Receive-before-remove applies to narrative custody too: Brain must land the canonical institution story before Engine compresses its duplicate.

## The Fix

Update Brain `README.md` only:

1. Add the institution depth currently stranded in Engine: named/persistent maintainer identities, cross-family review, wake/night-shift operation, transparent reasoning/memory, and the human gardener/merge gate.
2. Add a current roster derived from canonical identity/roster sources, or a durable roster link if a static table cannot be kept truthful.
3. Deepen the Native Edge Graph / DreamService / Golden Path explanation enough that Brain, not Engine, is the authoritative subsystem front door.
4. Replace every outward retired product-name occurrence with **Agent Institution** while preserving internal fleet vocabulary only where it names the actual subsystem.
5. Change the organization-map row to begin with the Brain repository link and end with plain **← You are here** outside the link; nothing precedes repository text.
6. Preserve the local logo/badges, install, Host/Cloud, MCP, dependency-transition, and learning doors already verified in PR #185.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| Brain institution story | canonical Introduction + identity registry + live Brain substrate | Brain owns the deep institution/night-shift explanation | Link to a canonical live roster rather than copy unverifiable rows | Brain README | source/roster audit |
| Agent OS mechanism story | Brain source + Brain guides | Native Edge Graph, A2A, DreamService and review are legible from the repo front door | Detailed guides remain canonical for internals | Brain README | mechanism/source mapping |
| organization map | operator formatting direction | Repository link first; current-location marker last and unlinked | none | Brain README | exact Markdown assertion |
| outward product name | D#17247 + operator ruling | Agent Institution outwardly; fleet only as subsystem vocabulary | disputed occurrence omitted pending classification | Brain README | occurrence/classification audit |

## Decision Record impact

Aligned-with ADR 0018, ADR 0040, D#17247, and the canonical Introduction. No ADR change.

## Acceptance Criteria

- [ ] Brain README contains the canonical institution/night-shift/cross-family-review story and no longer relies on Engine as the better explanation.
- [ ] Any roster/static identity fact is current and traceable to canonical identity sources; no stale model/handle table is copied blindly.
- [ ] Native Edge Graph, Memory Core, A2A, GitHub Workflow, DreamService/Golden Path, Neural Link, and the human merge gate form one coherent Brain story.
- [ ] Retired outward product-name occurrences are zero; Agent Institution is used outwardly, while internal fleet vocabulary is retained only for the subsystem.
- [ ] Every organization-map bullet begins with repository text; Brain alone ends with plain **← You are here** outside the link.
- [ ] Existing logo, badges, quickstart, operating modes, MCP, Engine-pin, learning, contribution, and license doors remain intact.
- [ ] All local Markdown/HTML targets exist and all new/changed public targets resolve.
- [ ] PR body records the exact Engine narrative rows that become removable only after this Brain receive merges.
- [ ] Cross-family identity review completes before merge.

## Out of Scope

Engine README removal · Agent Institution README · application/source/test changes · `package.json` · GitHub metadata · guide-tree work owned by #10 · Portal/SEO · product film replacement.

## Avoided Traps

- Deleting Engine's richer explanation before Brain receives it.
- Copying a stale roster table without source validation.
- Treating the retired outward product name as a global internal fleet refactor.
- Rewriting install/deployment material already validated by PR #185.

## Related

Prior Brain front door: #182 / PR #185  
Learning custody: #10  
Naming authority: D#17247  
Engine cleanup follows only after this receive merges.

Origin Session ID: d84687fa-f74c-466a-9c07-70e2efcc9be6

Retrieval Hint: `Brain README owns institution roster night shift Engine describes Brain better naming marker suffix`

Authored by Euclid (OpenAI GPT-5.6 Sol, Codex Desktop). Session d84687fa-f74c-466a-9c07-70e2efcc9be6.


## Timeline

- 2026-08-27T14:11:49Z @neo-gpt added the `documentation` label
- 2026-08-27T14:11:50Z @neo-gpt added the `enhancement` label
- 2026-08-27T14:11:50Z @neo-gpt added the `design` label
- 2026-08-27T14:11:50Z @neo-gpt added the `ai` label
- 2026-08-27T14:11:50Z @neo-gpt added the `agent-os` label
- 2026-08-27T14:12:26Z @neo-gpt assigned to @neo-gpt
- 2026-08-27T14:23:45Z @neo-gpt cross-referenced by #17813
- 2026-08-27T14:25:22Z @tobiu cross-referenced by PR #187
- 2026-08-27T15:01:43Z @neo-gpt-emmy cross-referenced by #195
- 2026-08-27T15:06:47Z @neo-gpt-emmy cross-referenced by #202
- 2026-08-28T07:29:14Z @tobiu referenced in commit `e97d8bc` - "Merge pull request #187 from neomjs/codex/186-brain-institution-story

docs(readme): make Brain own institution story (#186)"
- 2026-08-28T07:29:14Z @tobiu closed this issue

