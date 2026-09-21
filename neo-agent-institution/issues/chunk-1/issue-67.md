---
id: 67
title: Consume neo-agent-skills 0.1.3 — the decision-sweep and ownership-disposition rules
state: CLOSED
labels:
  - enhancement
  - ai
  - build
  - dependencies
assignees:
  - neo-opus-grace
createdAt: '2026-09-01T21:31:33Z'
updatedAt: '2026-09-01T21:43:08Z'
githubUrl: 'https://github.com/neomjs/neo-agent-institution/issues/67'
author: neo-opus-grace
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
closedAt: '2026-09-01T21:43:08Z'
---
# Consume neo-agent-skills 0.1.3 — the decision-sweep and ownership-disposition rules

## Context

`neo-agent-skills@0.1.3` published 2026-09-01, carrying `neomjs/neo-agent-skills#33` (resolves `#32` + `#34`). Two `ticket-create` rules land with it: §1a gains a Memory Core rationale arm and an own-assignment arm read by **bodies, not titles** (the three existing arms are all artifact substrates — they answer *"does a ticket exist?"*, never *"was this already decided, and why?"*), and §10 gains an exhaustive ownership disposition, because its assignment bullet gated on *"if you intend to start working immediately"* and a ticket filed as a **finding** never satisfied that antecedent.

## The Problem

This repository declares `neo-agent-skills: ^0.1.1` and its lockfile resolves **0.1.1**.

The range is not the binding constraint — `^0.1.1` already admits `0.1.3` under npm's `0.x` caret semantics. **The lockfile is**, and it pins the old tarball. Every install here reproduces the pre-`#33` corpus and projects `.claude/skills` from it, with nothing erroring: the skills materialize, CI passes, and the agent reads a `ticket-create` one version behind the one the swarm agreed on. A green install over a stale source — and because the corpus is *governance* substrate, the stale resolution degrades the rules agents file work against rather than a feature.

## The Architectural Reality

- `package.json` declares the range; `package-lock.json` decides the bytes. A range bump alone changes nothing here, and a lock refresh alone leaves the floor at a version that no longer carries the rules.
- Skills arrive as an npm dependency and are projected into `.claude/skills` as symlinks by the package's own postinstall linker; no skill bytes are committed, so *"did the projection happen"* is the only answerable question. `npm ci --ignore-scripts` yields a resolved dependency and zero reachable skills — resolution is not reachability.

## The Fix

Raise the declared floor to `^0.1.3` **and** refresh the lockfile in the same commit.

## Acceptance Criteria

- [ ] `package.json` declares `neo-agent-skills: ^0.1.3`.
- [ ] `package-lock.json` resolves `0.1.3` — verified by the `resolved` tarball URL, not by the range.
- [ ] `npx --no-install neo-agent-skills-materialize --check` passes at the new version, proving reachability rather than resolution.
- [ ] `.claude/skills` remains git-ignored.
- [ ] The materialized `ticket-create` payload carries the `(iii)`/`(iv)` sweep arms and §10's ownership disposition — `grep` the projected file, since a version number does not prove the content arrived.
- [ ] The repository's existing CI (`Explicit Brain contract`, `Isolated Institution`) stays green at the new resolution; a dependency bump under a postinstall linker is exactly the shape that can change what a test run sees.

## Out of Scope

- The skill payloads themselves; they are authored in `neomjs/neo-agent-skills` and consumed here.
- The sibling bumps in `neomjs/neo` and `neomjs/neo-agent-brain` — each has its own lockfile and CI, so each carries its own ticket.
- Retroactively auditing unassigned tickets against the new §10 rule.

## Avoided Traps

- **Bumping only the range** — `^0.1.1` already admitted `0.1.3`, so the install is unchanged and the ticket closes over nothing.
- **Bumping only the lock** — the floor would still permit a resolution without the rules.
- **Reading the version number as proof of content** — the consumer-visible artifact is the projected payload, not the manifest.
- **Hand-editing `package-lock.json`** — integrity hashes and the dependency graph belong to npm.

## Related

- `neomjs/neo-agent-skills#33` — the source PR
- Sibling consumer bumps: `neomjs/neo#18045` and `neomjs/neo-agent-brain`

**Live latest-open sweep:** latest 20 open issues in all three consumer repositories at 2026-09-01T21:29:49Z plus a `neo-agent-skills in:title` search across every state in each — no equivalent ticket. A2A claim sweep over the last 30 messages (all read-states, ~21:29 back to 20:00) shows no claim on this scope; `@neo-fable-clio`'s 21:00Z claim covers `#66` (dock witnesses), a disjoint surface.

**Structure-map gate:** N/A — manifest and lockfile only; no new `.mjs` placement.

**Ownership disposition:** assigned to `@neo-opus-grace`, who authored the upstream PR and is executing the three consumer bumps.

Origin Session ID: 5e4492c2-ace7-47e8-83bf-98ccca3a684b

Authored by Grace (Anthropic Claude Opus 5, Claude Code).

## Timeline

- 2026-09-01T21:31:33Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-01T21:31:35Z @neo-opus-grace added the `enhancement` label
- 2026-09-01T21:31:35Z @neo-opus-grace added the `ai` label
- 2026-09-01T21:31:35Z @neo-opus-grace added the `build` label
- 2026-09-01T21:31:36Z @neo-opus-grace added the `dependencies` label
- 2026-09-01T21:37:25Z @neo-opus-grace cross-referenced by PR #18046
- 2026-09-01T21:37:27Z @neo-opus-grace cross-referenced by PR #295
- 2026-09-01T21:37:30Z @neo-opus-grace cross-referenced by PR #68
- 2026-09-01T21:43:08Z @tobiu referenced in commit `c14c4c8` - "Merge pull request #68 from neomjs/grace/67-skills-013

chore(deps): consume neo-agent-skills 0.1.3 (#67)"
- 2026-09-01T21:43:09Z @tobiu closed this issue

