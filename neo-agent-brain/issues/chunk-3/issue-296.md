---
id: 296
title: The seat projector cannot bind a specifier whose package IS the target checkout
state: CLOSED
labels:
  - bug
  - ai
  - architecture
  - agent-os
assignees:
  - neo-opus-grace
createdAt: '2026-09-02T02:01:40Z'
updatedAt: '2026-09-02T09:49:35Z'
githubUrl: 'https://github.com/neomjs/neo-agent-brain/issues/296'
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
closedAt: '2026-09-02T09:49:35Z'
---
# The seat projector cannot bind a specifier whose package IS the target checkout

## Context

Measured during the nightshift heartbeat on 2026-09-02 01:53Z–02:10Z, in the Engine seat checkout on `dev`. This is **blocker 1** of the two-blocker stack analysed on #79; it is split out because it is a defect in the *hook projector*, not in arming coverage, and it is one-PR deliverable while #79's own ACs are all Codex-leg.

The observation that started it: three Claude seat hooks in the Engine checkout have been throwing on every session start since 2026-08-24, and **every surface stayed green**.

```
$ node .claude/hooks/wakeArmingHook.mjs
[WARN] [wake-arming] seat is UNARMED — wake arming threw: Cannot find package 'neo.mjs'
       imported from /Users/Shared/claude/neomjs/neo/.claude/hooks/wakeArmingHook.mjs
$ echo $?
0
```

## The Problem

`ai/scripts/lifecycle/hooks/projectSeatHooks.mjs` generates each seat's hooks by rewriting Brain-substrate specifiers to absolute paths beneath the runtime root, and **deliberately leaving package specifiers alone**. `rewriteSpecifiers`' contract states why: `neo.mjs/src/**` should resolve through the target's own `node_modules` against the published Engine, which is the ADR 0040 §2.3 dependency direction. Rewriting it would invent a dependency the source never declared.

That reasoning is correct and must survive this ticket. What the projector lacks is a notion of the **one target where the dependency direction degenerates**: the seat that *is* the package.

Node resolves a bare specifier from the importing file upward — `<engine>/.claude/hooks/` → `<engine>/node_modules/neo.mjs`. The Engine checkout's `package.json` is `name: "neo.mjs"`, so that directory cannot exist: a package does not carry itself in its own `node_modules`.

The failure is **Engine-seat-specific**, and one `ls` discriminates it:

| checkout | `node_modules/neo.mjs` | hook resolves? |
|---|---|---|
| `neo-agent-brain` | present | yes |
| `neo-agent-institution` | present | yes |
| `neo` (the Engine) | **absent, structurally** | **no** |

Brain- and institution-resident seats were never affected. (This corrects a broader claim I made on #79 at 01:02Z and broadcast; the correction is on that ticket.)

The reason it survived nine days unseen is the second half: the hook catches, logs `[WARN]`, and **exits 0**. To every consumer — the harness, CI, an operator — a dead hook and a working one are the same observation.

## The Architectural Reality

- `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs` — `ESM_SPECIFIER` matches only `./`- and `../`-relative specifiers in ESM specifier position, so bare package specifiers never enter `rewriteSpecifiers` at all. `renderProjection(source, runtimeRoot)` knows the runtime root and *not* the target root, so no layer in the render path can currently ask "what package is this target?".
- Six hook sources across all three harnesses carry the specifier: `claude/{wakeArmingHook,turnPresenceHook,laneStateStopHook}.mjs`, `codex/{codex-lane-state-stop,codex-context}.mjs`, `kimi-code/turnPresenceHook.mjs`. All six break in an Engine-seat projection.
- The Engine's `package.json` declares **no `exports` map**, so a subpath specifier *is* a file path beneath the package root and `<name>/src/Neo.mjs` → `<targetRoot>/src/Neo.mjs` is the identical resolution. That equivalence is a property of the current manifest, not a law, so the fix must not assume it silently.
- Related seam analysis (@neo-opus-vega, 2026-08-27, D#17644): these hooks need Brain policy and Engine runtime in one process, which is why the `neo.mjs` import exists at all and cannot simply be deleted.

## The Fix

A second, target-aware rewrite pass in `projectSeatHooks.mjs`, run after `rewriteSpecifiers` and never merged with it — the two answer different questions about different roots:

- `rewriteSelfPackageSpecifiers(contents, targetRepoRoot)` reads the target's own `package.json` `name`. When a specifier names that package, its subpath resolves against `targetRepoRoot`. For every target whose name differs, it is a no-op and §2.3 is untouched.
- `renderProjection` gains an optional `targetRepoRoot`; `checkProjection` and `projectHooks` already hold it.
- A target declaring an `exports` map is reported through the existing `escaped` channel rather than rewritten — same meaning (the projection cannot bind this specifier), same refusal in `projectHooks` — because a derived path would otherwise be a plausible-looking fabrication.

## Decision Record impact

`aligned-with ADR 0040`. §2.3's dependency direction is preserved verbatim for every target; this names the degenerate self-reference case §2.3 does not describe rather than weakening it. No ADR text changes.

## Acceptance Criteria

- [ ] A projection rendered for a target whose `package.json` `name` matches a specifier's package resolves that specifier inside the target.
- [ ] A target that merely *depends* on the package is unchanged — run as an explicit control, so an unconditional rewrite cannot pass.
- [ ] A self-package target declaring an `exports` map is reported, not guessed at.
- [ ] Red-first: the new arms must fail against the pre-fix source with the spec unchanged.
- [ ] The projected `wakeArmingHook.mjs` in an Engine-seat projection gets past module resolution — verified by executing it, not by reading its bytes.

## Out of Scope

- **Blocker 2 of #79: the Engine seat has no `fleet.planeBase`,** so it stays `UNARMED` after this fix — the failure moves from `Cannot find package 'neo.mjs'` to `fleet.planeBase is not configured`. That is an `ai/` config surface under the ADR-0019 gate (§critical_gates 10) and needs that ADR read properly, not a guess. Named here so this ticket cannot be closed on a green that means nothing.
- The Codex-leg arming coverage that #79's own ACs describe.
- The wider defect that a seat hook's only failure signal is prose in a `[WARN]` line that exits 0. Real, and the reason this survived nine days — but it may share a fix with blocker 2, so it is flagged rather than filed.

## Avoided Traps

- **Patching `rewriteSpecifiers` to rewrite package specifiers.** The obvious patch and the wrong one: it would invent a dependency for every seat to fix one, which is exactly what §2.3's carve-out exists to prevent.
- **A `node_modules/neo.mjs` self-symlink in the Engine checkout.** Works, untracked, invisible, and re-breaks on every clean install. It also puts provisioning state outside the projector's ledger, where nothing audits it.
- **A self-referential `imports`/`exports` entry in the Engine's own `package.json`.** Cheaper than the fix above, but it puts seat-projection concerns into the published package's manifest, where a consumer of `neo.mjs` inherits them.

## Related

- #79 — the parent finding; this is its blocker 1, Claude leg. Blocker 2 stays open there.
- #250 — restored and projected the harness hooks; this is the residual its projection could not have caught, because no target it was tested against was the package.
- #68 — the wake kill-switch's inoperative anti-flood layers: the same fails-open-and-green family, one layer up.

Live latest-open sweep: checked the latest 20 open issues in `neomjs/neo-agent-brain` at 2026-09-02T02:05Z plus a `projectSeatHooks OR specifier OR node_modules` all-state search and the `neomjs/neo-agent-skills` open queue; no equivalent found. A2A in-flight sweep over the last 30 messages: no competing `[lane-claim]` on the projector. Memory Core rationale sweep on the symptom's nouns surfaced @neo-opus-vega's 2026-08-27 seam measurement (corroborating, not duplicating) and no prior decision on the degenerate case. Own-assignment sweep: #79 is the only same-surface hit and this is its declared split.

Structure Map (§1c): N/A — no `.mjs` file is created or relocated; the change is confined to an existing module and its existing spec.

Origin Session ID: 24d75316-f075-429d-b168-a1ab0722a73a

Retrieval Hint: `query_raw_memories("projectSeatHooks self-package specifier Engine checkout is neo.mjs cannot find package unarmed")`, or `rewriteSelfPackageSpecifiers` / `ESM_SPECIFIER` in `ai/scripts/lifecycle/hooks/projectSeatHooks.mjs`.

🖖 Grace


## Timeline

- 2026-09-02T02:01:41Z @neo-opus-grace assigned to @neo-opus-grace
- 2026-09-02T02:01:42Z @neo-opus-grace added the `bug` label
- 2026-09-02T02:01:42Z @neo-opus-grace added the `ai` label
- 2026-09-02T02:01:43Z @neo-opus-grace added the `architecture` label
- 2026-09-02T02:01:43Z @neo-opus-grace added the `agent-os` label
- 2026-09-02T02:13:18Z @neo-opus-grace cross-referenced by PR #297
- 2026-09-02T09:49:36Z @tobiu referenced in commit `adf63b9` - "Merge pull request #297 from neomjs/fix/79-self-package-specifier-resolution

fix(lifecycle): the seat that IS the package resolves its own specifiers (#296)"
- 2026-09-02T09:49:36Z @tobiu closed this issue
- 2026-09-04T21:54:56Z @neo-opus-grace cross-referenced by #79
- 2026-09-04T23:12:26Z @neo-opus-grace cross-referenced by PR #320

