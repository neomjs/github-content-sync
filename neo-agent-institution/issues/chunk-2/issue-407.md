---
id: 407
title: 'Accounts: pick the repositories a seat gets clones for'
state: OPEN
labels:
  - enhancement
  - agent-os
  - ai
assignees:
  - neo-opus-ada
createdAt: '2026-10-01T18:41:34Z'
updatedAt: '2026-10-01T18:45:57Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/407'
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
blockedBy:
  - '[ ] 402 Brain pin 7: the installed FM carries resident placement, repo sets and the seat-home guard'
blocking: []
---
# Accounts: pick the repositories a seat gets clones for

## Context

The operator, 2026-09-30, on the first FM-launched peer (relayed on #245, comment 5913348117): *"this could be an FM enhancement: picking the repos (multiple ones) that a peer should get clones for."*

The Brain half has shipped. neomjs/neo-agent-brain#683 gives a seat other repositories (`metadata.repos`) and a `setRepos` verb, and clones each one with the seat's PAT before launch. #403 (Brain pin 7) brings it to the Institution. Every current peer works in several repositories: Ada's seat needs five.

## The Problem

The FM can set a seat's working repository (the add form's "Working repository", through `setRepo`), but nothing else. A seat's other repositories can only be set over the Fleet wire, so an operator can't give a seat the clones it needs.

## The Architectural Reality

- **Brain bridge:** `setRepos({id, repos})` (`ai/services/fleet/FleetControlBridge.mjs`, after `setRepo`) validates each entry with `setRepo`'s rules. It refuses a duplicate or the working slug, and `{id, repos: []}` clears the set. `listAgents()` returns the registry's definitions, `metadata` included.
- **Start outcome:** the start status carries `repos: [{repoSlug, state: 'prepared' | 'failed', reason?}]`. A failed entry's `reason` is credential-redacted and bounded.
- **Accounts card** (`AgentOS.view.fleet.detail.AgentConfigComponent`): it renders declared rows and sends edits as `configIntent` through the Panel to the bridge, as the harness and server chips do today.
- **In review:** neomjs/neo-agent-brain#710 / PR #711 adds a forge per repository. GitHub is the default; a GitLab entry carries an explicit clone URL and may name nested groups.

## The Fix

A "Repositories" section on the Accounts card:
- the working repository, plus the seat's other repositories as removable rows;
- an add field validated by the Brain's slug rule;
- each change sends one `setRepos` with the full list through the existing `configIntent` path, and a refusal shows its reason inline;
- the last start's per-repository outcome (prepared, or failed with its reason) beside each row, when the status carries it.

## Acceptance Criteria

- [ ] AC-1: the card lists the working repository and the seat's other repositories from the definition.
- [ ] AC-2: adding or removing a repository sends one `setRepos` with the whole list. A refusal (invalid slug, duplicate, the working slug) shows the Brain's reason, and the list keeps what the registry holds.
- [ ] AC-3: after a start, each repository shows its outcome, prepared or failed, with the redacted reason.
- [ ] AC-4: the section fits the card's measure at 1000×640 and 1280×800, with no clipped control. The goldens are re-captured at the 0.03 threshold.

## Out of Scope

- Per-harness write access to the extra checkouts, for example Codex's sandbox and OpenCode's `allowedPaths`. That's #682's recorded boundary, to be measured per harness.
- GitLab entries, which follow neomjs/neo-agent-brain#711 in its own leaf.
- Choosing the working repository after creation (`setRepo` stays with the add form).

## Intake (2026-10-01)

Self-authored this session, so only stage 2 runs (the prescription):
- **Prescription checked:** `apps/agentos/view/fleet/detail/AgentConfigComponent.mjs` owns the concern: the card's declared rows and its `configIntent`. `apps/agentos/view/accounts/Panel.mjs#onAgentConfigIntent` owns the bridge round-trip.
- **What the Fix implies on top of that, both measured in code:**
  - The round-trip today is the one `configureAgent` verb, but repositories travel by `setRepos`, so the Panel's intent handling routes a `repos` intent to that second verb.
  - `AgentOS.model.AgentDefinition` has no repository field, and nothing in the Institution maps `metadata.repo` / `metadata.repos` yet, so the definitions read has to carry them into the record.
- **The record shape:** a nested `metadata` field with `repo` and `repos` children. A field `mapping` won't do, because `RecordFactory` applies mappings only to initial values, so the round-trip's `record.set(agent)` readback would keep stale repositories. Nested fields cover both the `listAgents` hydration and the readback.
- **The refusal path (an open decision for AC-2):**
  - A `setRepos` refusal (invalid slug, duplicate, the working slug) is thrown in the Brain. It crosses the bridge as the sanitized generic error, because `installFleetBridge` passes only its bounded local errors verbatim.
  - `configureAgent`, by contrast, answers `{status: 'rejected', reason}` as data.
  - So "the Brain's reason" needs one of two things: either the Institution validates with the same rules before sending (the slug rule isn't in the Body-safe contract today), or the Brain's `setRepos` answers refusals as data (a Brain leaf). This gets decided at build time.
- **Placement:** sent to @neo-opus-grace as a Tier 2.5 fork. The recommendation is a sibling "Repositories" card under the config card at the same 28rem measure, because adding a repository needs a form field and the config card is a vdom render without one.

## Related

#245 · #403 (blocks: pin 7 carries `setRepos`) · neomjs/neo-agent-brain#682 / #683 · neomjs/neo-agent-brain#710 / #711 · neomjs/neo-agent-brain#571

## Sweeps

- Live latest-open sweep: the latest 20 open Institution issues, plus searches for "setRepos", "repo-set picker", "repository set" and "seat repositories". No equivalent.
- Memory Core: the multi-repo design discussion lives in #245's comments (Emmy, 09-30) and #682. There is no picker decision.
- A2A: no claim. Grace's #710 is the Brain forge model, adjacent.
- Own assignments: #402 and #404 (open, not overlapping).

Origin Session ID: 84a3bf84-c9cb-4215-818a-d9640f49669a

Authored by Ada (Claude Opus 5.5, Claude Code).



## Timeline

- 2026-10-01T18:41:34Z @neo-opus-ada assigned to @neo-opus-ada
- 2026-10-01T18:41:35Z @neo-opus-ada added the `enhancement` label
- 2026-10-01T18:41:36Z @neo-opus-ada added the `agent-os` label
- 2026-10-01T18:41:36Z @neo-opus-ada added the `ai` label
- 2026-10-01T18:41:40Z @neo-opus-ada marked this issue as being blocked by #402

