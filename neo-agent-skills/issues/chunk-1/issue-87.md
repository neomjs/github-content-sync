---
id: 87
title: 'session-sunset Step 1 runs a script the repo split moved, so every sunset outside the Brain errors'
state: OPEN
labels:
  - bug
  - ai
  - regression
  - agent-os
assignees: []
createdAt: '2026-09-17T20:19:43Z'
updatedAt: '2026-09-17T20:23:58Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/87'
author: neo-opus-vega
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
# session-sunset Step 1 runs a script the repo split moved, so every sunset outside the Brain errors

## Context

@neo-opus-grace hit this while executing `/session-sunset` in `neomjs/neo` on 2026-09-17, and reported it mid-run:

> *"Step 1's `--migrate-config` script doesn't exist in this repo anymore — the `ai/` tree moved to the Brain."*

[`session-sunset/references/session-sunset-workflow.md:65`](https://github.com/neomjs/neo-agent-skills/blob/dev/.agents/skills/session-sunset/references/session-sunset-workflow.md#L65) tells the sunsetting agent to run, relative to the clone it is sunsetting from:

```bash
node ai/scripts/setup/initServerConfigs.mjs --migrate-config
```

The `ai/` tree left `neomjs/neo` in `91ae3604b4` (2026-08-26, `#17791`). The skill is consumed from `node_modules/neo-agent-skills` in every org repo, so the step now runs where its script does not exist.

## The Problem

Measured across the org checkouts on 2026-09-17:

| repo | script | `config.template.mjs` | provisioned `config.mjs` overlay |
|---|---|---|---|
| `neo-agent-brain` | ✅ | ✅ | ✅ |
| **`neo`** | ❌ | ❌ | **✅** |
| `neo-agent-skills` | ❌ | ❌ | ❌ |
| `neo-agent-institution` | ❌ | ❌ | ❌ |
| `devindex` | ❌ | ❌ | ❌ |
| `create-app` | ❌ | ❌ | ❌ |

**The step is unrunnable outside the Brain by construction, not merely missing a file.** Its stated job is to *"reconcile the gitignored `config.mjs` operator-overlay with the pulled `config.template.mjs` leaves"*. Outside the Brain there is no template to reconcile against, so even vendoring the script back would not make the step meaningful.

Step 1 declares the refresh best-effort — *"must NEVER block sunset completion"* — so this has been degrading quietly rather than failing loudly. Every sunset in five of six repos since 2026-08-26 has logged a failure here, and the first person to say so out loud was a reviewer mid-handover.

**The same command appears a second time** at [`:102`](https://github.com/neomjs/neo-agent-skills/blob/dev/.agents/skills/session-sunset/references/session-sunset-workflow.md#L102), inside the linked-worktree staleness reminder, pointed at `<PRIMARY_ROOT>`. Whether that one is correct depends on which checkout `<PRIMARY_ROOT>` names post-split; it needs its own judgment rather than the same edit applied twice.

## The Architectural Reality

- The skill's own **Substrate Accretion Defense** note gives Step 1 an explicit retirement condition: it retires *"only if the clone topology itself changes (agents stop owning dedicated clones, or a session-liveness-aware sync lane ships)"*. **Neither happened.** What changed is where the script lives, so this is a relocation defect, not the retirement that note anticipated — the distinction matters, because retiring it silently would discard a rationale that is still sound for the Brain.
- `ai/config.mjs` and the per-server overlays under `ai/mcp/server/*/config.mjs` are **gitignored and provisioned per clone**. They are not repository artifacts, and their staleness is a property of an individual agent's checkout.
- In the `neo` clone used for this measurement, those six overlays are all dated **2026-07-18**, while the Brain's template moved on **2026-08-28**. That is consistent with no reconcile having run since before the split — though it is one clone, and a per-clone file proves nothing about anyone else's.

## The Fix

**Step 1: make the step stop firing where it cannot work.** The minimal form is a guard rather than a deletion, so the Brain keeps the behaviour its own rationale still justifies:

```bash
if [ -f ai/scripts/setup/initServerConfigs.mjs ]; then
    node ai/scripts/setup/initServerConfigs.mjs --migrate-config
fi
```

Deleting the line outright is the alternative and is defensible — but it removes the refresh from the one repo where it is correct, so whoever takes this should state which they chose and why.

**Step 2: decide the `:102` occurrence separately**, once `<PRIMARY_ROOT>`'s post-split meaning is settled.

**Step 3 — ANSWERED after filing, by @tobiu, and the answer simplifies this ticket.**

This originally asked whether `neo`'s provisioned overlays needed a reconcile path, and parked it as ADR-0019-gated. Two facts closed it:

1. **ADR-0019 moved to the Brain as well**, so the entire AiConfig concern is Brain-scoped. There is no `ai/` config question left in the engine repo to gate.
2. **The overlays configure nothing.** Measured in one clone after the ruling: `ai/` held **6 files, all `config.mjs`** — no server, no client, no executable of any kind. The servers left with the split; only their config stayed behind. They are orphans, not live state.

So the step is Brain-only, full stop. `neo`'s overlays needed no reconcile path because they had no consumer. Deleting `ai/` entirely from an engine clone is correct and safe — @tobiu confirmed it, and it is gitignored per-clone state either way. One knock-on worth noting for whoever edits: `.claude/settings.local.json` carries a permission entry for `ai/mcp/client/mcp-cli.mjs`, which the same split already made dead.

## Acceptance Criteria

- [ ] **AC-1** — a sunset run in a repo without the script completes Step 1 without an error line, proven by running it in `neomjs/neo` rather than by reading the diff.
- [ ] **AC-2** — a sunset run in `neo-agent-brain` still performs the reconcile, proven by observing the script execute. A change that silences the error by removing the behaviour everywhere fails this.
- [ ] **AC-3** — the `:102` occurrence is either corrected or explicitly recorded as correct, with `<PRIMARY_ROOT>`'s post-split referent named.
- [ ] **AC-4** — the Substrate Accretion Defense note is updated to say the step is Brain-scoped, so the next reader does not re-derive why it is guarded and mistake the guard for the documented retirement.
- [x] ~~**AC-5** — the unowned question in Step 3 above is filed as its own ticket or answered in this one's thread.~~ **Answered before work started** — see Step 3: ADR-0019 is Brain-hosted too, and the engine clone's `ai/` held only orphaned config with no consumer. Nothing to own.

## Out of Scope

- **Changing what `initServerConfigs.mjs` does**, or where `ai/` lives.
- **Any `ai/` config authoring.** It lives in the Brain now, ADR-0019 included, so it is not merely gated from here — it is not in this repo's territory at all.
- **The rest of the sunset workflow.** Only Step 1 and the `:102` reminder are in question.
- **Auditing other agents' clone overlays.** They are gitignored per-clone state, not substrate.

## Avoided Traps

- ⛔ **Do not delete the step as "dead".** Its rationale is alive in the Brain and carries a documented retirement condition that has not been met. What is dead is its *location assumption*.
- ⛔ **Do not vendor the script into each repo.** Without a `config.template.mjs` there is nothing to reconcile against, so a copied script would run and do nothing — a worse failure than the current loud one, because it would look like success.
- ⛔ **Do not infer fleet-wide drift from one clone's file dates.** The overlays are per-clone; the 2026-07-18 timestamps above describe one checkout.
- ⛔ **Do not treat "best-effort, never blocks" as "safe to leave".** That is exactly the property that let this run unreported for three weeks across five repos.

## Related

- `neomjs/neo#17791` / `91ae3604b4` — the split that moved `ai/` to the Brain
- `#40` — `neo-agent-brain` calls no PR baseline; same family of post-split substrate fallout

`unowned-rationale:` parked, and now a smaller job than when filed — Step 3 is answered, so what remains is a skill-text edit plus the `:102` judgment. @neo-opus-grace surfaced it at 0% budget and is going dark; I am mid-lane on the engine side. Any seat can take it; a Brain-side one can confirm AC-2 without switching repos.

Origin Session ID: 2b78af80-54a8-4e14-afd8-85c6dfe41004

Retrieval Hint: "session-sunset migrate-config initServerConfigs missing after repo split, ai tree moved to Brain, config.template.mjs absent outside Brain"


## Timeline

- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

