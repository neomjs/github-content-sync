---
id: 21
title: Hooks ride the skills materializer instead of a config-string merge
state: CLOSED
labels:
  - bug
  - ai
  - regression
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-08-30T16:09:22Z'
updatedAt: '2026-08-30T21:06:51Z'
githubUrl: 'https://github.com/neomjs/neo-agent-skills/issues/21'
author: neo-opus-grace
commentsCount: 3
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
closedAt: '2026-08-30T21:06:51Z'
---
# Hooks ride the skills materializer instead of a config-string merge

## Context

Measured 2026-08-30 on a live Claude seat: the `Stop` hook enforcing §L3_No_Hold has been crashing on every turn end since 2026-08-28, failing **open**. A deference fallback that the gate exists to catch went through unchallenged, and @tobiu — not the gate — caught it.

The diagnosis and its controls are on neomjs/neo-agent-brain#250. This ticket is the **prescription**, and it is deliberately not the one #250 originally proposed. @tobiu's steer: *"non repo specific ci workflows and hook should work the same way. we already started moving items over here => and installers."*

Live latest-open sweep: checked all 11 open issues in this repo at 2026-08-30T16:07:45Z; A2A claim sweep over the last 30 messages (all read-states) shows no overlapping `[lane-claim]`. No equivalent found.

## The Problem

Engine commit `c623b2f63c` (neomjs/neo#17791 / PR neomjs/neo#17806) deleted three hook entrypoints — `laneStateStopHook.mjs` (983 lines), `turnPresenceHook.mjs` (126), `wakeArmingHook.mjs` (156) — and stripped the hooks block from the Engine's tracked `.claude/settings.template.json`.

The seat's `.claude/settings.json` is **gitignored and machine-local**, so it kept declaring all four hook events. Running the Stop hook's exact configured command:

```
Error: Cannot find module '.../.claude/hooks/laneStateStopHook.mjs'
```

Control: `rgReplaceGuardHook.mjs`, the one surviving script, same path template and node — exit 0.

The transport is why this was silent. Hooks are wired as **command strings naming a filesystem path**, materialized by `mergeClaudeHooks` (Brain `ai/scripts/setup/initServerConfigs.mjs:1254`):

```js
const mergedHooks = {...(activeSettings.hooks || {}), ...templateHooks};
```

A one-way spread. It *ensures* template hooks and *preserves* everything else permanently — no retirement path. A path that stops existing produces a process exiting 1, which the harness treats as non-blocking. Nothing anywhere asserts the target exists.

## All three harnesses, three different failure modes

`c623b2f63c` did not strand Claude alone — it deleted every harness's entrypoints. Reproduced on a second seat at a different checkout by @neo-opus-ada, and read from the commit stat:

| harness | hook config | scripts | resulting state |
|---|---|---|---|
| Claude | gitignored, **survived** the cut | deleted | dangling references → exit-1 crash, fail-open |
| Codex | `.codex/hooks.json` was tracked, **deleted with them** | deleted | clean absence — nothing declared |
| Kimi | **generated fresh** by the Brain's `ai/services/fleet/generateKimiSeatConfig.mjs:243` | deleted | every newly generated seat is born pointing at a deleted script |

The Kimi row is still *producing* the defect rather than preserving it. So the projection must be harness-agnostic with per-harness opt-in — which is the manifest shape this package already has, not a new capability.

## The Architectural Reality

This repository already solved this exact class, for skills, in `scripts/materialize-harness-skills.mjs`. Hooks are not a new transport problem — they are a surface the existing transport does not yet cover.

The materializer projects the installed package as **untracked, gitignored symlinks whose targets live under `node_modules`**, copying zero bytes into consumer git, and its `--check` arm already asserts precisely the two properties a string-path hook config cannot:

| materializer check | line | the hook failure it would have caught |
|---|---|---|
| `is a dangling link; its node_modules target is gone` | `:179` | the deleted `laneStateStopHook.mjs` |
| `is not projected by the manifest — an invented or stale entry` | `:192` | the four stale hook events on every pre-split seat |

It also already writes into `.claude/` — `FACADE = '.claude/skills'`, per-skill links precisely because a manifest may opt a skill out of the Claude façade. `.claude/hooks` is the same shape with the same opt-in need.

The JSDoc names our failure mode in our own words: *"a consumer with the dependency resolved and no links materialized — the invisible-absence failure... `--check` is the arm that makes it loud: CI asserts materialization succeeded rather than assuming the hook ran."* That is the hook bug, described before it happened, on a sibling surface.

`.github/workflows/reusable-pr-baseline.yml` establishes the same ownership direction for CI, and #14 states it as policy: *"`neo-agent-skills` becomes the source of truth for non-product PR governance... Consumer repositories commit only minimal pinned callers."* Harness hooks are the runtime sibling of that statement — non-product, cross-repository, and currently Engine-local by accident of history rather than design.

## The Fix

Extend the existing transport to a third surface rather than building a second one:

1. **A `hooks/` surface in this package** holding the harness-neutral entrypoints, with the manifest governing per-harness projection (a Codex or Kimi seat does not want the Claude Stop hook).
2. **`materialize-harness-skills.mjs` projects `.claude/hooks`** as per-entry symlinks, exactly as it does `.claude/skills`, and `--check` covers them with the same dangling/stale/tracked assertions it already applies.
3. **Consumers wire hook commands at the projected path**, so a retired hook becomes a dangling link that `--check` fails loudly in CI — instead of an exit-1 crash nobody sees.

The three orphaned entrypoints are recoverable from `c623b2f63c^`.

## Contract Ledger Matrix

| Target Surface | Source of Authority | Proposed Behavior | Fallback | Docs | Evidence |
|---|---|---|---|---|---|
| `neo-agent-skills-materialize` | `scripts/materialize-harness-skills.mjs` | additionally projects `.claude/hooks` per manifest | none — absence is a `--check` failure, never silent | the script's `@summary` block | `--check` red on a removed hook target |
| `skills.manifest.json` | same package | gains per-harness hook projection entries | a hook absent from the manifest is not projected | manifest schema docs | `test-materialize` arm |
| `.claude/hooks/*` in consumers | this package via symlink | untracked projection, never tracked bytes | refuse if tracked, as skills already do | consumer READMEs | existing `tracked()` refusal at `:186` |

## Decision Record impact

`depends-on ADR 0035` (Live Lane Awareness, Accepted 2026-07-12). ADR-0035 `:33` names `.claude/hooks/laneStateStopHook.mjs` as a consumer of the fenced hook projection; that consumer currently has no implementation. This ticket restores an implementation without changing the contract.

## Acceptance Criteria

- [ ] The harness hook entrypoints live in this package and are projected into `.claude/hooks` as untracked symlinks by `materialize-harness-skills.mjs`.
- [ ] `--check` fails on a hook whose target is missing, on a projected hook absent from the manifest, and on a tracked hook path — verified by mutating each of the three and observing red.
- [ ] The manifest governs per-harness projection, so a non-Claude seat is not given the Claude façade hooks.
- [ ] A consumer whose hook was retired from the manifest no longer carries it after materialization — the retirement path the string-merge lacks.
- [ ] Migration note records that a consumer's hook command must name the projected path.
- [ ] **Landing this repairs source, not seats.** Every seat's `.claude/settings.json` is gitignored and machine-local, so a seat that has not re-run the materializer still carries dead hooks and is still fail-open after this merges. The rollout is verified per seat — a report naming which seats have been re-materialized — not inferred from the merge. Closing this ticket while any seat still resolves a hook command to a non-existent script is a false close.

## Out of Scope

- **Whether the Stop hook's enforcement is restored, softened, or retired.** That is the §L3 no-hold teeth — Tier-4 operator authority. This ticket is transport only: it makes any hook's absence loud. The behavioural disposition, and neomjs/neo-agent-brain#124's false-positive findings, are separate.
- Retiring the Brain's `mergeClaudeHooks` / `initClaudeSettings` Claude slice and the zero-caller `stopHookDecision.mjs` — neomjs/neo-agent-brain#250 owns that tail, gated on this landing.
- CI workflow unification (#14 owns that half of @tobiu's "same way").

## Avoided Traps

- **Adding a retirement path to the Brain's string-merge.** That was #250's original prescription and it is the wrong one: it re-derives, worse, the dangling-target and stale-entry checks this package already ships, and leaves two mechanisms where one belongs. The substrate error came from sweeping neo and the Brain but not this repository.
- **Copying hook bytes into each consumer.** The operator rejected that twice for skills as an SSOT violation; the objection is structural and applies identically here.
- **Treating the deleted entrypoints as a deliberate retirement.** Turn presence *was* re-homed (to the Brain's MCP path, still reporting `presenceTerminal: completed`) while the other two were not, and the Brain kept the 439-line `stopHookDecision.mjs` with zero callers. The asymmetry says incomplete extraction, not decision.

## Related

- neomjs/neo-agent-brain#250 — the measured diagnosis and the Brain-side tail.
- #14 — the CI half of the same ownership move; this is its runtime sibling.
- neomjs/neo-agent-brain#124 — Stop-hook false positives; premise unobservable while the script is absent.
- neomjs/neo-agent-brain#79 — session-start wake arming; neomjs/neo#16410 shipped it 2026-08-24, the split deleted the hook 2026-08-28.
- neomjs/neo#17791 / PR neomjs/neo#17806 — the split that deleted the entrypoints.

## Handoff Retrieval Hints

Retrieval Hint: "harness hooks ride the skills materializer symlink transport instead of a config-string merge"
Retrieval Hint: Engine commit `c623b2f63c`; `scripts/materialize-harness-skills.mjs:179` and `:192`

Origin Session ID: 7fc710e4-f796-47bc-97b7-b61d1414b3a3


## Timeline

- 2026-08-30T16:09:24Z @neo-opus-grace added the `bug` label
- 2026-08-30T16:09:24Z @neo-opus-grace added the `ai` label
- 2026-08-30T16:09:24Z @neo-opus-grace added the `regression` label
- 2026-08-30T16:09:25Z @neo-opus-grace added the `architecture` label
- 2026-08-30T16:09:25Z @neo-opus-grace added the `agent-os` label
- 2026-08-30T16:10:10Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-08-30T16:10:10Z @neo-opus-grace cross-referenced by #250
- 2026-08-30T17:53:59Z @neo-gpt-emmy cross-referenced by #22
- 2026-08-30T18:03:13Z @neo-gpt-emmy cross-referenced by #14
- 2026-08-30T18:04:43Z @neo-gpt-emmy cross-referenced by #23
### @neo-opus-grace - 2026-08-30T20:52:57Z

## Prescription falsified before implementation — the hooks are Brain-coupled by construction

I started this lane by recovering the entrypoints from `c623b2f63c^` and checking their imports before copying anything. All nine recover cleanly. None of them is portable to this package.

Every entrypoint reaches into `ai/**` by relative path:

| entrypoint | imports |
|---|---|
| `.claude/hooks/laneStateStopHook.mjs` | `../../ai/scripts/lifecycle/parseLaneState.mjs`, the deference-directive builder |
| `.claude/hooks/turnPresenceHook.mjs` | `../../ai/mcp/server/memory-core/helpers/TurnPresenceHookWriter.mjs` |
| `.claude/hooks/wakeArmingHook.mjs` | `../../ai/daemons/wake/armSeatWakeRoute.mjs`, `readSubscriptionsOverMcp.mjs` |
| `.codex/hooks/codex-lane-state-stop.mjs` | `../../ai/scripts/lifecycle/parseLaneState.mjs`, `classifyPromptingContext` |
| `.codex/hooks/codex-context.mjs` | the same `TurnPresenceHookWriter` |
| `.kimi-code/hooks/turnPresenceHook.mjs` | `TurnPresenceHookWriter`, `../../ai/graph/normalizeAgentIdentityNodeId.mjs` |
| `.kimi-code/hooks/wakeEnvelopeHook.mjs` | `../../ai/graph/normalizeAgentIdentityNodeId.mjs` |

Verified at head: all six of those modules exist in `neomjs/neo-agent-brain`, and this package has **no `ai/` tree at all**.

So hosting the entrypoints here would require either duplicating the Brain's `ai/**` — the exact SSOT violation `scripts/materialize-harness-skills.mjs` rejects in its own header, and which the operator rejected twice for skills — or inventing a cross-package resolution that does not exist today. Neither is worth doing to satisfy a placement I inferred.

### Where the inference went wrong

@tobiu's steer was *"non repo specific ci workflows and hook should work the same way. we already started moving items over here => neo-agent-skills => and installers."* That names the **mechanism** — installer-projected, untracked symlinks, a `--check` arm — and I read it as a **destination**. #14 is consistent with the narrower reading: this package is the SSOT for *non-product PR governance*, reusable workflows and the policy tools they invoke. Harness hooks are runtime wiring, not PR governance.

### Proposed reshape

**The hooks are restored in the Brain**, where their dependencies already live and the relative imports work unmodified — a straight restore rather than a port. **The Brain gains an installer that follows this package's proven pattern**: untracked symlink projection with a `--check` arm that fails on a dangling target, a stale unprojected entry, or a tracked path. That is the operator's "same way", satisfied by the mechanism rather than by the address.

This package keeps what it demonstrably owns: the transport pattern itself, `reusable-pr-baseline.yml`, and the skill corpus.

### Disposition

Retitling and re-scoping this ticket to own the **pattern export** — what a consuming repo must implement to project hooks the way skills are projected — with the Brain-side restore tracked on neomjs/neo-agent-brain#250, whose scope already covers that repo's hook materialization. #250 stays blocked on this only for the pattern contract, not for a file move.

Recording it here rather than quietly refiling: the ticket was mine, the prescription was mine, and the evidence that killed it took four commands. Third prescription this session falsified by checking rather than by reasoning — the ADR-0019 lesson generalises well past `ai/` config.


### @neo-opus-grace - 2026-08-30T21:06:49Z

## Closing — ADR 0040 §2.7 already owns this, and it rules against the placement I proposed

@neo-gpt-emmy's self-correction sent me to ADR 0040, which I should have read before filing. It settles every question this ticket opened, and not in this ticket's favour.

**§2.7, verbatim:** lane-state, wake, presence and Memory-Core context hooks under `.claude/hooks`, `.codex/hooks` and `.kimi-code/hooks` are **Agent OS substrate**; **their sources move by exact-identity census** — never a blanket directory move, with Engine-only contributor guards that have no Brain dependency staying Engine-owned; and **seat provisioning materializes the moved hooks into target checkouts as generated-not-tracked artifacts**, with the Engine's ignore rules taking the generated paths. **Leaf 11 owns that scope.**

**§2.5** adds the resolution rule: tracked seat hooks are the third consumer family and resolve Brain substrate **only through `agentosRuntimeRoot`-provisioned artifacts, never relatively from `targetRepoRoot`**.

So the ADR already specifies destination, census discipline, projection mechanism, resolution authority and ignore rules. This ticket re-derived all five, and got the first one wrong: hooks are Agent OS substrate, not skills-package content. The census rule also retroactively explains the one survivor — `rgReplaceGuardHook.mjs` is an Engine-only contributor guard with no Brain dependency, so it correctly stayed.

**The defect is real and better named by the ADR than by me:** leaf 6 executed and leaf 11 did not. Hook sources left the Engine; nothing materializes them into target checkouts. That is a sharper root than the retirement-path gap I originally wrote, and it explains the three-way asymmetry exactly — Claude's gitignored config survived the cut and dangles, Codex's tracked config left with the sources, and Kimi's generator predates leaf 11 and still emits a path no repository contains.

**Disposition:** closing as superseded. neomjs/neo-agent-brain#250 is re-anchored on §2.5/§2.7 and carries the whole lane, including @neo-gpt-emmy's second dependency leg — six entrypoints also lazily import Engine `src/`, which per §2.3 must become package-qualified against the published Engine, so the restore is not a byte-for-byte replay.

Nothing here belongs to this repository. `reusable-pr-baseline.yml` and the skill corpus remain its own; the projection *pattern* stays demonstrated by `materialize-harness-skills.mjs` and needs no ticket to be borrowed.

**What I'd keep from this:** I filed a ticket for a contract an accepted ADR already owned, because I inferred a placement from an operator's mechanism steer instead of reading the record that governs the edit. Two peers then corrected it in sequence — Emmy on the dependency leg, the ADR on the placement. The cheap move I skipped was four minutes of `grep` in `learn/agentos/decisions/`.


### @neo-opus-grace - 2026-08-30T21:06:50Z

Superseded by ADR 0040 §2.5/§2.7 and neomjs/neo-agent-brain#250. Reasoning in the preceding comment.

- 2026-08-30T21:06:51Z @neo-opus-grace closed this issue
- 2026-09-04T20:43:07Z @neo-opus-vega cross-referenced by #51
- 2026-09-18T12:19:32Z @neo-opus-vega cross-referenced by #90

