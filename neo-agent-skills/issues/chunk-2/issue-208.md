---
id: 208
title: 'overview skill: pass sections: [''focus''] to get_sandman_handoff'
state: OPEN
labels:
  - enhancement
  - ai
  - model-experience
  - agent-os
assignees: []
createdAt: '2026-10-25T04:19:36Z'
updatedAt: '2026-10-25T04:19:36Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/208'
author: neo-gemini-julia
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
# overview skill: pass sections: ['focus'] to get_sandman_handoff

## Context

[`neomjs/neo-agent-brain#1251`](https://github.com/neomjs/neo-agent-brain/issues/1251) (implemented in [`neomjs/neo-agent-brain#1269`](https://github.com/neomjs/neo-agent-brain/pull/1269)) changes `get_sandman_handoff`'s default `sections` parameter to `['goldenPath']` so a bare boot call returns only the `## Computed Golden Path` slice (~3.5 KB vs ~32 KB). In her review of `neomjs/neo-agent-brain#1269` (`PRR_kwDOUBzDFM8AAAABUEHWZQ`, RA-3), `@neo-opus-grace` identified that the `overview` skill (`skills/overview/references/overview-read.md:23`) calls `get_sandman_handoff` bare and then inspects the response for `## Focus (v1)`.

## The Problem

While [`neomjs/neo-agent-brain#957`](https://github.com/neomjs/neo-agent-brain/issues/957) has not yet shipped the `## Focus (v1)` section in the Sandman handoff, once `#957` lands, a bare `get_sandman_handoff` call will return only `sections.goldenPath` (`content: null`) and omit `sections.focus`. As a result, `overview-read.md:23` would print `Focus (v1): absent from the handoff` even when `## Focus (v1)` is present in the handoff file.

## The Architectural Reality

- `skills/overview/references/overview-read.md:23` currently instructs:
  `1. Call get_sandman_handoff (see its description for the freshness envelope).`
- Under `neomjs/neo-agent-brain#1251` / `#1269`, `get_sandman_handoff` accepts `sections: ['focus']` (returning `{sections: {focus: {heading, content}}, availableSections, missingSections}`) and `sections: 'all'`.
- `skills/overview/SKILL.md` is the Progressive Disclosure router and needs no change.

## The Fix

Update `skills/overview/references/overview-read.md:23` to call `get_sandman_handoff({sections: ['focus']})` and read `sections.focus` (checking `missingSections` / `sections.focus === null` for the absent branch).

## Contract Ledger

| Target surface | Source of authority | Proposed behavior | Fallback / edge | Docs | Evidence |
|---|---|---|---|---|---|
| `skills/overview/references/overview-read.md:23` (`## 1. The section first`) | `neomjs/neo-agent-brain#1251` (`get_sandman_handoff` `sections` contract) + `neomjs/neo-agent-brain#957` (`## Focus (v1)`) | Call `get_sandman_handoff({sections: ['focus']})` and read `sections.focus.content` when non-null | When `sections.focus` is `null` (`'focus'` in `missingSections`), print `Focus (v1): absent from the handoff (as of <envelope time>)` and continue with `§3` | `skills/overview/references/overview-read.md` | `npm test` in `neo-agent-skills` |

Decision Record impact: none (aligns with ADR 0008 Progressive Disclosure; router `SKILL.md` untouched).

## Acceptance Criteria

- [ ] AC-1 `skills/overview/references/overview-read.md` §1 step 1 calls `get_sandman_handoff({sections: ['focus']})` and step 2 reads `sections.focus` (`sections.focus === null` triggers the existing absent branch).
- [ ] AC-2 `skills/overview/SKILL.md` remains untouched (Progressive Disclosure router budget preserved).

## Out of Scope

Implementing the `## Focus (v1)` producer in the Sandman handoff (`neomjs/neo-agent-brain#957`).

## Avoided Traps

Passing `sections: 'all'` in `/overview` when only `## Focus (v1)` is needed — `sections: ['focus']` keeps the `/overview` handoff read bounded to the Focus slice and returns `sections.focus: null` cleanly before `#957` lands.

## Related

- Companion to [`neomjs/neo-agent-brain#1251`](https://github.com/neomjs/neo-agent-brain/issues/1251) / [`neomjs/neo-agent-brain#1269`](https://github.com/neomjs/neo-agent-brain/pull/1269) (`RA-3` by `@neo-opus-grace`)
- Producer ticket: [`neomjs/neo-agent-brain#957`](https://github.com/neomjs/neo-agent-brain/issues/957)

unowned-rationale: One-line companion leaf in `neo-agent-skills` to land alongside or before `neomjs/neo-agent-brain#957`; parked unowned while finishing the active `neo-agent-brain` PR review loop (`#1269` / `#1270`).

Live latest-open sweep: checked latest 20 open issues in `neomjs/neo-agent-skills` (`#206`..`#14`) at `2026-10-25T04:18:53Z`; no equivalent found.
A2A in-flight claim sweep: checked latest 30 messages via `list_messages({status: 'all', limit: 30})` at `2026-10-25T04:18:33Z`; no competing claim on `overview-read.md`.
MC rationale sweep: `query_raw_memories({query: 'overview get_sandman_handoff focus sections'})` at `2026-10-25T04:18:39Z`; no prior decision against `sections: ['focus']`.
Own-assignment sweep: `gh issue list --repo neomjs/neo-agent-skills --state open --assignee @me` returned `0` open issues.
Structure map: `skills/overview/references/overview-read.md` — existing reference payload in `neomjs/neo-agent-skills`; no new file.

Origin Session ID: 2d131263-cb25-4f19-8a19-98ef961cc86b
Retrieval Hint: "overview-read get_sandman_handoff sections focus #1251 #957"

## Timeline

- 2026-10-25T04:19:38Z @neo-gemini-julia added the `enhancement` label
- 2026-10-25T04:19:38Z @neo-gemini-julia added the `ai` label
- 2026-10-25T04:19:38Z @neo-gemini-julia added the `model-experience` label
- 2026-10-25T04:19:38Z @neo-gemini-julia added the `agent-os` label
- 2026-10-25T04:25:24Z @neo-gemini-julia cross-referenced by PR #1269

