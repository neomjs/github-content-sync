---
id: 211
title: Skills gain an app audience for workspaces built on neo
state: CLOSED
labels:
  - enhancement
  - ai
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-10-25T23:56:48Z'
updatedAt: '2026-10-26T01:29:34Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/211'
author: neo-opus-grace
commentsCount: 0
parentIssue: 20114
subIssues: []
subIssuesCompleted: 0
subIssuesTotal: 0
contentTrust:
  projected: true
  quarantined: 0
  signals: []
blockedBy: []
blocking:
  - '[x] 45 neo-app writes the agent layer into the workspaces it creates'
closedAt: '2026-10-26T01:29:34Z'
---
# Skills gain an app audience for workspaces built on neo

## Context

This is the skills leaf of neomjs/neo#20114, graduated from neomjs/neo D#20113 (S1, plus Julia's STEP_BACK consumer partial). A workspace that `neo-app` creates for an outside developer should get only the skills that help someone build an app on neo. That set is not our maintainer workflow.

## The Problem

Every consumer gets every skill. The skills are written for maintainers of our repositories: reviews, tickets, roles, the lifecycle gates. In an app workspace they cost router context, and their triggers fire maintainer gates inside someone else's repository. Nothing teaches an agent how to author a neo app.

## The Architectural Reality

- `scripts/materialize-harness-skills.mjs:43-54`: `.agents/skills` is **one directory symlink** to the package's skills, the harness-neutral surface read by Codex and other harnesses. `.claude/skills` is **per-skill links**, filtered by the manifest's `claudeSymlinkRequired`. So a whole-directory link cannot filter by audience.
- `.agents/skills/skills.manifest.json` has per-skill rows and no audience field.
- `scripts/generate-agents-md.mjs:10`: "Applicability is DECLARED, never inferred". Sections declare repositories and audiences, and only `maintainer` and `contributor` exist today.
- `tracked()` (`materialize-harness-skills.mjs:111`) returns `false` when `git` fails, so a workspace that is not a git repository is handled. The AC below keeps that true.
- The `neural-link` skill is a router plus `references/operational-handbook.md`.

## The Fix

1. **Manifest:** an optional per-skill `audiences` list (schema and validation). A row without it keeps today's projection.
2. **Materializer:** `--audience <name>`. When it is declared, both `.agents/skills` and `.claude/skills` become per-skill links to exactly the skills whose rows list that audience. Without the flag, the projection in our repositories stays exactly as it is today.
3. **Generator:** an `app` audience for the declared `neo-app-workspace` repository kind: `app` sections and an `app` preamble, emitting the `AGENTS.md` for an app workspace. It is declared, never inferred from the consumer, and every existing refusal still applies.
4. **Skills:**
   - `neo-app-authoring` is new: a router `SKILL.md` plus a references payload (ADR 0008). It is distilled from the engine's app-authoring rules:
     - read a sibling and `src/Neo.mjs`, `src/core/Base.mjs`, `src/state/Provider.mjs`, `src/data/Model.mjs` and `src/data/Store.mjs` before app code;
     - start from the matching engine primitive;
     - data UI binds a `data.Store` of `data.Model` records;
     - providers stay at view roots, and styles stay in SCSS;
     - look classes up through `https://neomjs.com/llms.txt` and `/raw/**`.
   - `neural-link` lists `app`, and its start instruction works in an app workspace: the server comes from the workspace's MCP config, not from `npm run ai:*`.
5. **Version bump** in the PR.

## Contract Ledger

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `skills.manifest.json` row `audiences` | `skills.manifest.schema.json` | optional list of audience names per skill | absent = today's projection | schema, README | schema spec |
| `neo-agent-skills-materialize --audience <name>` | `materialize-harness-skills.mjs` | per-skill links on both surfaces for the declared audience | no flag = today's projection, unchanged | script header | spec: the `app` projection equals the `app` rows; the no-flag projection is unchanged |
| `neo-agent-skills-agents-md --repo neo-app-workspace --audience app` | `generate-agents-md.mjs` | emits the app `AGENTS.md` for the declared workspace kind | an unknown audience or repository is still refused | script header, README | spec |

## Acceptance Criteria

- [ ] With `--audience app`, `.agents/skills` and `.claude/skills` each hold exactly the skills whose rows list `app`, as per-skill links, and `--check` passes.
- [ ] It also works in a directory that is not a git repository.
- [ ] Without `--audience`, the projection in our repositories is unchanged (a spec compares it).
- [ ] The generator emits an `app` `AGENTS.md` from declared `app` sections and preamble, for the declared `neo-app-workspace` kind.
- [ ] `neo-app-authoring` exists as a router plus payload (ADR 0008) and lists `app`.
- [ ] `neural-link` lists `app`, and its start instruction holds in an app workspace.
- [ ] The package version is bumped in the PR.

## Out of Scope

- `neo-app`'s use of the audience (its own leaf under neomjs/neo#20114).
- Any `app` skill beyond these two.

## Related

Refs neomjs/neo#20114 · graduated from neomjs/neo D#20113 · adjacent: `#144` (how a release reaches consumers).

Sweeps: live latest-open (20, `2026-10-25T23:52:52Z`), A2A (30, `23:53Z`), MC (1 problem-noun query) and own-assignment (none here) found no equivalent.

Origin Session ID: 5597537c-fa0a-4b88-8847-edddb7492a11

Retrieval Hint: `query_raw_memories("skills app audience materializer per-skill links agents/skills neo-app-authoring")`


## Timeline

- 2026-10-25T23:56:49Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-10-25T23:56:50Z @neo-opus-grace added the `enhancement` label
- 2026-10-25T23:56:50Z @neo-opus-grace added the `ai` label
- 2026-10-25T23:56:50Z @neo-opus-grace added the `agent-os` label
- 2026-10-25T23:57:11Z @neo-opus-grace added parent issue #20114
- 2026-10-25T23:57:15Z @neo-opus-grace marked this issue as blocking #45
- 2026-10-26T00:29:23Z @neo-opus-grace cross-referenced by PR #212
- 2026-10-26T00:42:22Z @neo-opus-grace referenced in commit `ecfaf74` - "fix(skills): neo-app-authoring's description within the 220-byte description budget of #210 (#211)"
- 2026-10-26T00:57:28Z @neo-opus-grace referenced in commit `48e3db1` - "feat(skills): an app audience for workspaces built on neo (#211)

`neo-agent-skills-materialize --audience app` links only the skills whose
manifest row lists `app`, on both surfaces as per-skill links: a directory
link at `.agents/skills` would expose the maintainer corpus to an outside
developer's harness. The audience is declared by the caller and never
inferred; a row without `audiences` serves maintainers, so the projection
without the flag is unchanged for every existing row.

The v1 app set is `neural-link` and a new `neo-app-authoring` skill (router
plus payload, ADR 0008). The generator emits the matching instruction file for
the declared `neo-app-workspace` kind (`--repo neo-app-workspace --audience
app`), with its own preamble and two sections and no maintainer gate.

New contract test for the materializer (both projections, the switch, the
refusals, run outside a git repository); red on 3e98245, where `--audience`
is silently ignored. Version 0.1.60."
- 2026-10-26T00:57:29Z @neo-opus-grace referenced in commit `eb79714` - "fix(skills): neo-app-authoring's description within the 220-byte description budget of #210 (#211)"
- 2026-10-26T00:57:30Z @neo-opus-grace referenced in commit `30cdbd5` - "fix(skills): release 0.1.61, after #210 shipped 0.1.60 (#211)"
- 2026-10-26T01:01:39Z @neo-opus-grace referenced in commit `1940247` - "fix(skills): release 0.1.61, after #210 shipped 0.1.60 (#211)

[skill-growth-justified: the corpus grows by the app audience, two skills that only a workspace built on neo projects; a maintainer's Claude façade loads neither new byte, and the authoring payload is the one guide an outside agent needs to write engine-shaped code]"
- 2026-10-26T01:29:34Z @tobiu referenced in commit `a146e23` - "feat(skills): an app audience for workspaces built on neo (#211) (#212)

* feat(skills): an app audience for workspaces built on neo (#211)

`neo-agent-skills-materialize --audience app` links only the skills whose
manifest row lists `app`, on both surfaces as per-skill links: a directory
link at `.agents/skills` would expose the maintainer corpus to an outside
developer's harness. The audience is declared by the caller and never
inferred; a row without `audiences` serves maintainers, so the projection
without the flag is unchanged for every existing row.

The v1 app set is `neural-link` and a new `neo-app-authoring` skill (router
plus payload, ADR 0008). The generator emits the matching instruction file for
the declared `neo-app-workspace` kind (`--repo neo-app-workspace --audience
app`), with its own preamble and two sections and no maintainer gate.

New contract test for the materializer (both projections, the switch, the
refusals, run outside a git repository); red on 3e98245, where `--audience`
is silently ignored. Version 0.1.60.

* fix(skills): neo-app-authoring's description within the 220-byte description budget of #210 (#211)

* fix(skills): release 0.1.61, after #210 shipped 0.1.60 (#211)

[skill-growth-justified: the corpus grows by the app audience, two skills that only a workspace built on neo projects; a maintainer's Claude façade loads neither new byte, and the authoring payload is the one guide an outside agent needs to write engine-shaped code]"
- 2026-10-26T01:29:35Z @tobiu closed this issue

